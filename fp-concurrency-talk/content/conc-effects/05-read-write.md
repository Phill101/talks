## ReadWrite

```scala
trait ReadWrite[F[_], A] {

  /** current value and release action*/
  def read:  F[(A, F[Unit])]

  /** current value and putAndRelease action*/
  def write: F[(A, A => F[Unit])]

}
```


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


```scala
override def read: F[(A, F[Unit])] = 
  Deferred[F, (A, F[Unit])].bracketCase { d =>
    state.modify {
      case s @ State(q, r, false, a) if !q.exists(_.isWriter) => 
        s.copy(currentReaders = r + 1) -> d.complete(a -> releaseRead)
      case s @ State(q, _, _, _) => s.copy(q = q :+ Reader(d)) -> F.unit
    }.flatten *> d.get
  } {
    case (d, ExitCase.Canceled | ExitCase.Error(_)) =>
      state.update(s => s.copy(q = s.q.filterNot {
        case Reader(df) => df == d
        case _ => false
      }))
    case _ => F.unit
  }
```
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
      state.update(s => s.copy(q = s.q.filterNot {
        case Writer(df) => df == d
        case _ => false
      }))
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
