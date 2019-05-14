<div style="align-items: center; display: flex;">
    <pre style="margin: 10px" class="fragment fade-right stretch" data-fragment-index="1"><div class="fragment" data-fragment-index="2">Thread1</div><code class="scala" data-trim data-noescape>
class Auction(startsWith: Bid) {
  private var highest: Bid = startsWith

  def currentBidder: Long = highest.userId

  def makeBid(bid: Bid): Unit =
    if <span class="fragment highlight-green-then-dark-green" data-fragment-index="3">(bid.amount > highest.amount)</span> <span class="fragment highlight-green-then-dark-green" data-fragment-index="6">highest = bid</span>
}
	</code></pre>
    <pre style="margin: 10px" class="fragment fade-left stretch" data-fragment-index="2"><div>Thread2</div><code class="scala" data-trim data-noescape>
class Auction(startsWith: Bid) {
  private var highest: Bid = startsWith

  def currentBidder: Long = highest.userId

  def makeBid(bid: Bid): Unit =
    if <span class="fragment highlight-green-then-dark-green" data-fragment-index="4">(bid.amount > highest.amount)</span> <span class="fragment highlight-green-then-dark-green" data-fragment-index="5">highest = bid</span>
}
	</code></pre>
</div>


```scala
class Auction(startsWith: Bid) {
  private var highest: Bid = startsWith

  def currentBidder: Long = highest.userId

  def makeBid(bid: Bid): Unit = this.synchronized {
    if (bid.amount > highest.amount) highest = bid
  }
}
```
- <!-- .element: class="fragment" data-fragment-index="1" -->Блокирует тред
- <!-- .element: class="fragment" data-fragment-index="2" -->Может привести к дедлоку
- <!-- .element: class="fragment" data-fragment-index="3" -->Тяжело рассуждать
- <!-- .element: class="fragment" data-fragment-index="4" -->Не композируется

Note: Тяжело прерывать уже залоченные треды.


```scala
class AuctionActor(startsWith: Bid) extends Actor {
  var highest: Bid = startsWith

  override def receive: Actor.Receive = {
    case GetCurrentBidder => sender() ! highest.userId
    case MakeBid(b: Bid)  => 
      if (b.amount > highest.amount) highest = b
  }
}

object AuctionActor {
  case object GetCurrentBidder
  case class  MakeBid(bid: Bid)
}
```
- <!-- .element: class="fragment" data-fragment-index="1" -->Тяжело тестировать и дебажить
- <!-- .element: class="fragment" data-fragment-index="2" -->Тяжело рассуждать

Note: Акторы не только тяжело тестировать и дебажить из-за принципа взаимодействия между акторами и отсутствия кол стэка, акторы так же могут создать новые акторы, меня поведение существующих.


```scala
class Auction(startsWith: Bid) {
  private val highestRef: AtomicReference[Bid] = new AtomicReference(startsWith)

  def currentBidder: Long = highestRef.get.userId

  def makeBid(bid: Bid): Unit = highestRef.updateAndGet { highest =>
    if (bid.amount > highest.amount) bid
    else highest
  }
}
```
- <!-- .element: class="fragment" data-fragment-index="1" -->Side-effectful

Note: Похоже что java.util.concurrent неплох и может решать наши проблемы. Что на счёт ещё какого-нибудь concurrency примитива, например Semaphore?


```scala
import java.util.concurrent.Semaphore

val semaphore = Semaphore(3)
semaphore.acquire(2)
semaphore.acquire(2)
```
- <!-- .element: class="fragment" data-fragment-index="1" -->Мы можем лучше

Note: Что здесь произойдёт? Залочится тред... операционной системы... Звучит нехорошо. Может мы можем лучше? Да. Мы можем.
Все эти рассмотренные случаи объединяет императивный подход. 
