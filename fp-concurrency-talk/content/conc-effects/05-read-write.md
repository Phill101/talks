## ReadWrite

```scala
trait ReadWrite[F[_], A] {

  /** current value and release action*/
  def read:  F[(A, F[Unit])]

  /** current value and putAndRelease action*/
  def write: F[(A, A => F[Unit])]

}
```
```scala
val io = for { 
  rw <- ReadWrite.of[IO, Int](10) 
  (ten1, release1) <- rw.read 
  _  <- rw.write.timeout(1.second).attempt 
  (ten2, release2) <- rw.read 
  _  <- release1
  _  <- release2
} yield (ten1, ten2)
```
<!-- .element: class="fragment" data-fragment-index="1" -->

Note: Можно взять блокировку в рид режиме, либо в врайт режиме.
Пока в очереди нет врайтеров, ридеры свободно могут брать и отпускать.
Когда появляется хотя бы один врайтер, он ложится в очередь ждать, пока все ридеры отпустят, и все остальные ридеры и врайтеры после него тоже отправляются в очередь.
Нужно не забыть, что F[A] может быть отменён.


<pre><code class="scala" data-trim data-noescape>
final case class State\[F\[\_\], A\](q: Queue[<span class="fragment" data-fragment-index="1">Op\[F, A\]</span>\],
                                currentReaders: Int, 
                                writeLocked: Boolean, 
                                value: A)
</code></pre>
```scala
sealed trait Op[F[_], A] {
  def isReader: Boolean = false
  def isWriter: Boolean = false
}

final case class Reader[F[_], A](d: Deferred[F, (A, F[Unit])]) extends Op[F, A] {
  override def isReader = true
}

final case class Writer[F[_], A](d: Deferred[F, (A, A => F[Unit])]) extends Op[F, A] {
  override def isWriter = true
}
```
<!-- .element: class="fragment" data-fragment-index="1" -->


```scala
object ReadWrite {

  final class ConcurrentReadWrite[F[_], A](state: Ref[F, State[F, A]])
                                 (implicit F: Concurrent[F]) extends ReadWrite[F, A] {

    override def read : F[(A, F[Unit])] = ???

    override def write: F[(A, A => F[Unit])] = ???

    private val releaseRead: F[Unit] = ???

    private def releaseWrite(a: A): F[Unit] = ???
  }

  def of[F[_]: Concurrent, A](value: A): F[ReadWrite[F, A]] = 
     Ref.of[F, State[F, A]](State(Queue.empty, 0, false, value))
       .map(new ConcurrentReadWrite[F, A](_))

}
```


<pre><code class="scala" data-trim data-noescape>
override def read: F[(A, F[Unit])] = 
  Deferred[F, (A, F[Unit])]<span class="fragment" data-fragment-index="0">.<span class="fragment fade-in" style="position:absolute" data-fragment-index="7">bracketCase { d =></span><span class="fragment fade-out" data-fragment-index="7">flatMap { d =></span>
    <span class="fragment" data-fragment-index="1">state.modify {
      <span class="fragment" data-fragment-index="2">case s @ State(q, r, false, a) if q.isEmpty =></span>
        <span class="fragment" data-fragment-index="3">s.copy(currentReaders = r + 1) -> d.complete(a -> releaseRead)</span>
      <span class="fragment" data-fragment-index="4">case s @ State(q, _, _, _) => </span><span class="fragment" data-fragment-index="5">s.copy(q = q :+ Reader(d)) -> F.unit</span>
    }</span><span class="fragment" data-fragment-index="3">.flatten</span><span class="fragment" data-fragment-index="6"> *> d.get</span>
  } </span><span class="fragment" data-fragment-index="7">{
    <span class="fragment" data-fragment-index="8">case (d, ExitCase.Canceled | ExitCase.Error(_)) =></span>
      <span class="fragment" data-fragment-index="9">state.update(s => s.copy(q = s.q.filterNot(_ == Reader(d))))</span>
    <span class="fragment" data-fragment-index="10">case _ => F.unit</span>
  }</span>
</code></pre>

```scala
private val releaseRead: F[Unit] = 
  state.modify {
    case s @ State(q, r, _, _) if q.isEmpty => 
      s.copy(currentReaders = r - 1) -> F.unit
    case s @ State(Writer(d) +: tl, 1, _, a) => 
      s.copy(q = tl, currentReaders = 0, writeLocked = true) -> 
        d.complete(a -> releaseWrite)
    case s @ State(Writer(_) +: _, r, _, _) => 
      s.copy(currentReaders = r - 1) -> F.unit
    case State(Reader(_) +: _, _, _, _) => 
      sys.error("wrong state. next in queue should be writer")
  }.flatten
```
<!-- .element: class="fragment" data-fragment-index="11" -->


```scala
override def write: F[(A, A => F[Unit])] = 
  Deferred[F, (A, A => F[Unit])].bracketCase { d =>
    state.modify {
      case s @ State(q, 0, false, a) if q.isEmpty => 
        s.copy(wl = true) -> d.complete(a -> releaseWrite)
      case s @ State(q, _, _, _) => 
        s.copy(q = q :+ Writer(d)) -> F.unit
    }.flatten *> d.get
  } {
    case (d, ExitCase.Canceled | ExitCase.Error(_)) =>
      state.update(s => s.copy(q = s.q.filterNot(_ == Writer(d))))
    case _ => F.unit
  }
```
```scala
private def releaseWrite(a: A): F[Unit] = 
  state.modify {
    case s @ State(q, _, _, _) if q.isEmpty  => 
      s.copy(value = a, wl = false) -> F.unit
    case s @ State(Writer(d) +: tl, _, _, _) => 
      s.copy(q = tl, value = a) -> d.complete(a -> releaseWrite)
    case s @ State(q, _, _, _) =>
      val nextReaders = q.iterator.takeWhile(_.isReader).collect {
        case Reader(d) => d.complete(a -> releaseRead)
      }.toList
      val rCount = nextReaders.size

      s.copy(q = q.drop(rCount), value = a, wl = false, cr = rCount) ->
        nextReaders.foldRight(F.unit)(_ *> _)
  }.flatten
```
<!-- .element: class="fragment" data-fragment-index="1" -->


```scala
def get: F[A] = read.flatMap { 
  case (a, release) => release.as(a)
}

def set(a: A): F[A] = update(_ => a)
def update(f: A => A): F[Unit] = modify(a => (f(a), ()))
def modify(f: A => (A, B)) = write.flatMap {
  case (a, putAndrelease) =>
    val (u, b) = f(a)
    putAndrelease(u).as(b)
}

def tryRead: F[Option(A, F[Unit])]
def tryGet: F[Option[A]]
...
```
