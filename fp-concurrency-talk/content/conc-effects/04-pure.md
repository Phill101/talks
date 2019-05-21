```scala
abstract class Ref[F[_], A] {
  def get: F[A]
  def set(a: A): F[Unit]
  def update(f: A => A): F[Unit]
  def modify[B](f: A => (A, B)): F[B]
  // ... and more
}

object Ref {
  def of[F[_]: Sync, A](a: A): F[Ref[F, A]]
}
```
```scala
for {
  ref <- Ref.of[IO, Int](42)
  v   <- ref.get
  _   <- putStrLn(s"value is: $v") // 'value is: 42'
  old <- ref.modify(x => ((x + 1), x + "!"))
  nu  <- ref.get
  _   <- putStrLn(s"$old and $nu") // '42! and 43'
} yield ()
```
<!-- .element: class="fragment" data-fragment-index="1" -->


```scala
class Auction private(highestRef: Ref[IO, Bid]) {
  def currentBidder: IO[Long] = highestRef.get.map(_.userId)

  def makeBid(bid: Bid): IO[Unit] = highestRef.update { highest =>
    if (bid.amount > highest.amount) bid
    else highest
  }
}

object Auction {
  def startWith(bid: Bid): IO[Auction] =
    Ref.of[IO, Bid](bid).map(new Auction(_))
}
```

Note: "Теперь, имея необходимые знания мы наконец можем увидеть и понять решение функционального подхода. И это решение похоже на то, что мы уже видели ранее с AtomicReference."
"Можно заметить причудливую инициализацию через объект компаньон..."


```scala
def putStrLn(s: Any): IO[Unit] = IO(println(s))

val io: IO[Unit] = for {
  semaphore <- Semaphore[IO](3)
  f <- semaphore.acquireN(2).start
  _ <- semaphore.acquireN(2).timeout(1.second).start
  _ <- IO.sleep(1.second) *> semaphore.available >>= putStrLn
} yield ()
```


```scala
abstract class Deferred[F[_], A] {    
  def get: F[A]    
  def complete(a: A): F[Unit]
}
```
```scala
for {    
  d <- Deferred[IO, Int]
  f <- d.get.flatTap(i => putStrLn(s"completed with $i")).start
  _ <- d.complete(42)
  i <- f.join
} yield i
```
<!-- .element: class="fragment" data-fragment-index="1" -->
