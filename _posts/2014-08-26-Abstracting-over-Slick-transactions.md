---
layout: post
title: Abstracting over Slick transactions
date: 2014-08-26
categories: 
- development
tags: scala Slick transactions
author: oyvind
---


Different needs will necessarily lead down different paths in terms of
application architecture and a multitude of abstractions to support them.
Abstractions at this level might help for example reduce boilerplate,
enable looser coupling between modules, or enable wanted design patterns.

One of the things we really appreciate about using Slick for the
persistence layer is how everything is explicit and type-checked.
This is especially true for transaction handling, which (completely
ignoring `` Database.withDynSession()`` ) is all checked compile time.

Another thing that we like about Slick, is that unless you really resist,
it naturally leads you towards using the cake pattern to wire your
application. The combination of designing an application by modules
with the full expressive power of the Scala type system, and the
compile-time sanity checks of database handling gives us a lot
of flexibility and strong guarantees.

This blog post is about one of the challenges in that very intersection
that we solved by identifying one small abstraction.


Cake pattern and Slick
----------------------

Normally one might declare that a given module has a dependency on a
database by extending a trait like this, in which you encapsulate
the database driver and connection:

{% highlight scala %}
trait DbModule {
  val profile: Slick.driver.JdbcDriver
  val db:      profile.simple.Database
}
{% endhighlight %}

A module in the persistence layer, then, might look something like this:

{% highlight scala %}
trait FooPersistingModule extends DbModule {
  def persist(f: Foo) = db withSession { implicit s =>
    ??? //save to database
  }
}
{% endhighlight %}

Loose coupling between layers (abstract modules)
----------------------------------------------

A typical way of accomplishing this is to simply extract an
abstract module, and have all client modules use that:

{% highlight scala %}
trait FooRepoModule {
  def fooRepo: FooRepo
  trait FooRepo{
    def save(f: Foo): Unit
    def lookup(id: FooId): Option[Foo]
  }
}

trait FooRepoDbModule extends FooRepoModule with DbModule {
  object fooRepo extends FooRepo {
    def save(f: Foo) = db withSession (implicit s => ???)
    def lookup(id: FooId) = ???
  }
}
{% endhighlight %}

By having the layer above depend only on the
abstract ``FooRepoModule``, we achieve a certain decoupling
of business logic from implementation details.

The reason behind this insistence upon decoupling is both
a matter of principle (we like it), and more importantly
practical: It is way easier to test the logic in the
service layer without needing to setup a database.


Orchestrating transactions
--------------------------

The way we wrote the abstract module unfortunately impedes us
from controlling transactions at a higher layer in the application,
which is bad from both a performance and a semantic point of view.

{% highlight scala %}
//will use fs.size number of transactions
def saveFoos(fs: Foo*) = fs foreach fooRepo.save
{% endhighlight %}

It is clear that to fix that problem, the methods
in ``FooRepo`` need to take a Slick ``Session`` as a parameter:

{% highlight scala %}
trait FooRepoNotSoAbstractModule extends DbModule {
  def fooRepo: FooRepo
  trait FooRepo {
    def save(f: Foo)(implicit s: profile.simple.Session): Unit
    def lookup(id: FooId)(implicit s: profile.simple.Session): Option[Foo]
  }
}
{% endhighlight %}

This enables us to write a service layer module like the following:

{% highlight scala %}
trait FooServiceDbModule extends FooRepoNotSoAbstractModule with DbModule {
  def saveFoos(fs: Foo*) = db withSession (implicit s => fs foreach fooRepo.save)
}
{% endhighlight %}

The problem again then, however, is that both modules inherit ``DbModule``.
There are of course a number of stopgap solutions here, for example
to move saveFoos() down-layer to ``FooRepoModule``, or to write an abstract
``FooServiceModule`` containing most of the logic.
However, once your service module needs to call two different repositories
in the same transaction, such workarounds tend to get in the way.


An abstraction over transactions
--------------------------------

This problem led us to the very simple abstraction of ``TransactionAware``,
something armed only with the knowledge of the existence of transactions.

{% highlight scala %}
trait TransactionAware {
  type Tx
}
{% endhighlight %}

With that knowledge, an abstract repository might now look like this

{% highlight scala %}
trait FooRepoTxModule extends TransactionAware {
  def fooRepo: FooRepo
  trait FooRepo{
    def save(f: Foo)(implicit tx: Tx): Unit
    def lookup(id: FooId)(implicit tx: Tx): Option[Foo]
  }
}
{% endhighlight %}

To cover the two cases, with and without Slick, consider
the following two implementations of ``TransactionAware``:

{% highlight scala %}
trait SlickTransactionAware
  extends TransactionAware {
  self: SlickProfile =>

  final override type Tx = self.profile.simple.Session
}


trait DummyTransactionAware extends TransactionAware {
  implicit object DummyTransaction
  final override type Tx = DummyTransaction.type
}
{% endhighlight %}

The way they both override ``Tx`` for the whole application
they are wired into, means that you have to provide
exactly one of them. ``DummyTransactionAware`` has
no dependency on Slick.

We mentioned testing as one of the wanted advantages,
so let's see what a test repository module backed by
a ``mutable.Map<A, B>`` looks like:

{% highlight scala %}
trait FooRepoTestingModule extends FooRepoTxModule with DummyTransactionAware {
  object fooRepo extends FooRepo {
    private val foos = collection.mutable.Map[FooId, Foo]()
    def save(f: Foo)(implicit tx: Tx) = foos(f.id) = f
    def lookup(id: FooId)(implicit tx: Tx) = foos.get(id)
  }
}
{% endhighlight %}

So far we abstracted over the knowledge of the existence of transactions.
This demonstrates how to abstract over the ability to work with transactions:

{% highlight scala %}
trait TransactionBoundary
  extends TransactionAware {

  def transaction: Transaction

  trait Transaction {
    def readOnly[A](f: Tx => A): A
    def readWrite[A](f: Tx => A): A
  }
}
{% endhighlight %}

This is how we can rewrite ``FooServiceDbModule`` from earlier in terms of the
new abstraction, this time independent of Slick.

{% highlight scala %}
trait FooServiceModule extends FooRepoTxModule with TransactionBoundary {
  def saveFoos(fs: Foo*) = transaction readWrite (implicit tx => fs foreach fooRepo.save)
}
{% endhighlight %}


Final wiring
------------

Analogously with the ``SlickTransactionAware`` and ``DummyTransactionAware`` traits,
there are two implementations of ``TransactionBoundary``:
The surprisingly named ``SlickTransactionBoundary`` and ``DummyTransactionBoundary``.

When ultimately concretizing the abstract modules, we provide the proper
``TransactionBoundary``. Of course, the compiler will complain if it encounters
``DummyTransactionAware`` in an application wired with ``SlickTransactionBoundary``,
and vice versa. Real abstract modules, ie. those that only implement
``TransactionAware``, can obviously be mixed in for both scenarios.

{% highlight scala %}
class FooServiceTest
  extends FooServiceModule
  with FooRepoTestingModule
  with DummyTransactionBoundary {
    //tests
  }

class FooApplication
  extends FooServiceModule
  with FooRepoDbTxModule //Not shown
  with SlickTransactionBoundary
  with DbModule {
    val profile = ???
    val db      = ???
  }
{% endhighlight %}



Complete source code can be found [here](https://github.com/pengerno/slick-transactions/), including a [buildable example application](https://github.com/pengerno/slick-transactions/blob/master/testing-liquibase/src/test/scala/no/penger/db/TransactionsTest.scala ).

