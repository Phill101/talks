## STM


```scala
def transfer(from  : Ref[Account], 
             to    : Ref[Account], 
             amount: Amount): IO[Unit] = ???
  // if (from.get >= amount) 
  //   from.update(_ - amount) *> to.update(_ + amount)
  // else 
  //   IO.sleep(1.second) *> transfer(from, to, amount)
```


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

- <!-- .element: class="fragment" data-fragment-index="1" --> Atomicity
- <!-- .element: class="fragment" data-fragment-index="2" --> Consistency
- <!-- .element: class="fragment" data-fragment-index="3" --> Isolation

Note: Доступ к данным осуществляется внутри транзакций. Находясь внутри транзакции, поток не видит изменений, производимых другими потоками. Сохранение совершенных изменений происходит при выходе из транзакции. Если в этот момент выясняется, что данные были изменены другим потоком, транзакция начинает выполняться сначала. Если внутри транзакции будет брошено исключение, данные не будут изменены. 
1) STM дает нам атомарность (будет изменено все или ничего)
2) Согласованность (данные согласованы до начала транзакции и остаются таковыми после ее завершения)
3) Изолированность (не видно изменений, производимых другими потоками)
Другими словами, мы получаем буквы A, C и I из ACID. Также транзакционная память интересна тем, что, в отличие от традиционных мьютексов и семафоров, она обеспечивает отсутствие дэдлоков в наших программах.


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
