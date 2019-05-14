## STM


```scala
def transfer(from  : TRef[Account], 
             to    : TRef[Account], 
             amount: Amount): STM[Unit] =
  for {
    balance <- from.get
    if balance >= amount // Or STM.check(balance >= amount)
    _ <- from.update(_ - amount)
    _ <- to.update(_ + amount)
  } yield ()
```
```scala
def moneyTransfer(sender   : User, 
                  recipient: User, 
                  amount   : Amount): IO[Unit] =
  for {
    from <- sender.account
    to   <- recipient.account
    _    <- transfer(from, to, amount).commit[IO]
  } yield ()
```


```scala
class TSemaphore private(val permits: TRef[Long]) {
  def acquireN(n: Long) =
    for {
      value <- permits.get
      if value >= n
      _ <- permits.set(value - n)
    } yield ()

  def releaseN(n: Long) =
    permits.update(_ + n)
}

object TSemaphore {
  def make(n: Long) = TRef(n).map(new TSemaphore(_))
}
```
