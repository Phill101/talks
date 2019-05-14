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

Note: "Теперь, имея необходимые знания мы наконец можем увидеть и понять решение функционального подхода. Это решение похоже на то, что мы уже видели ранее с AtomicReference. Только здесь мы используем чистый аналог Ref, предоставляемый системой эффектов. Можно заметить причудливую инициализацию через объект компаньон. Это связано с треобваниями ссылочной прозрачности. Это очевидное и натуральное решение (если немного подумать), я не буду на нём подробно останавливаться."


```scala
def putStrLn(s: Any): IO[Unit] = IO(println(s))

val io: IO[Unit] = for {
  semaphore <- Semaphore[IO](3)
  f <- semaphore.acquireN(2).start
  _ <- semaphore.acquireN(2).timeout(1.second).start
  _ <- IO.sleep(1.second) *> semaphore.available >>= putStrLn
} yield ()
```

Note: Ну хорошо, а что с семафором? Пояснить за код.
