---
title: Zionomicon chapters 6 - 9
description:
author: Jim Weinert
keywords: scala
url: https://github.com/jwcarvana
marp: true
---

# Zionomicon: Parallelism and Concurrency

An exploration of the concurrency building blocks in ZIO.

---

# Concurrency is not Parallelism

![width:800px](/assets/rob-pike.png)
"Concurrency is dealing with a lot of things at once.
Parallelism is doing a lot of things at once."
- Rob Pike, https://vimeo.com/49718712

---

# Chapter 6: The Fiber Model

---

# Fibers vs Threads

| | Fibers | Threads
---|:---:|:---:
Max # | hundreds of thousands | limited by operating system
Interruptable | ✅ | ⚠️
Typed | ✅ | 🚫
Joinable | ✅ | 🚫

---

# Types

An **Effect** describes a single ZIO[-R, +E, +A] execution.

The **Fiber** is created by the ZIO runtime to execute a program. It does not immediately execute.

The **Executor** is a thread pool that runs a ZIO program one execution at a time.

The **maximumYieldOpCount** is the number of instructions to run before context switching.

---

# Chapter 7: Concurrency Operators

---

# Operations

## Fork

```scala
import zio._

trait ZIO[-R, +E, +A] {
    def fork: URIO[R, Fiber[E, A]]
}
```


---

# Operations

## Fork

```scala
lazy val example1 =
    for {
        _ <- doSomething
        _ <- doSomethingElse
    } yield ()
```


---

# Operations

## Fork

```scala
lazy val example2 =
    for {
        _ <- doSomething.fork
        _ <- doSomethingElse
    } yield ()
```

---

# Operations

## Join

```scala
trait Fiber[+E, +A] {
    def join: IO[E, A]
}
```

---

# Operations

## Join


```scala
lazy val example2 =
    for {
        fiber <- doSomething.fork
        result <- fiber.join
    } yield result
```
---

# Operations

## Await

```scala
trait Fiber[+E, +A] {
    def await: UIO[Exit[E, A]]
}
```

---

# Operations

## Poll

```scala
trait Fiber[+E, +A] {
    def poll: UIO[Option[Exit[E, A]]]
}
```


---

# Operations

## Interrupt

```scala
trait Fiber[+E, +A] {
    def interrupt: UIO[Exit[E, A]]
    // Cause.Interrupt
}
```

---

# Operations

## Fiber supervision model

1. Every fiber has a scope
2. Every fiber is forked in a scope
3. Fibers are forked in the scope of the current fiber unless otherwise specified
4. The scope of a fiber is closed when the fiber terminates, either through
success, failure, or interruption
5. When a scope is closed all fibers forked in that scope are interrupted

---

# Operations

## Fork Daemon

```scala
import zio._

trait ZIO[-R, +E, +A] {
    def forkDaemon: URIO[R, Fiber[E, A]]
}
```

---

# Locking

```scala
import scala.concurrent.ExecutionContext

trait ZIO[-R, +E, +A] {
    def lock(executor: Executor): ZIO[R, E, A]
    def on(executionContext: ExecutionContext): ZIO[R, E, A]
}
```

---

# Locking

```scala
lazy val doSomething: UIO[Unit] = ???
lazy val doSomethingElse: UIO[Unit] = ???

lazy val executor: Executor = ???

lazy val effect =
    for {
        _ <- doSomething.fork
        _ <- doSomethingElse
    } yield ()

lazy val result = effect.onExecutor(executor)
```

---

# Locking

```scala
lazy val doSomething: UIO[Unit] = ???
lazy val doSomethingElse: UIO[Unit] = ???

lazy val executor1: Executor = ???
lazy val executor2: Executor = ???

lazy val effect2 =
    for {
        _ <- doSomething.onExecutor(executor2).fork
        _ <- doSomethingElse
    } yield ()

lazy val result = effect2.onExecutor(executor1)
```

---

# Concurrency Operators

---

# Concurrency Operators

## Race and ZipPar

```scala
trait ZIO[-R, +E, +A] { self =>
    def raceEither[R1 <: R, E1 >: E, B](
        that: ZIO[R1, E1, B]
    ): ZIO[R1, E1, Either[A, B]]

    def zipPar[R1 <: R, E1 >: E, B](
        that: ZIO[R1, E1, B]
    ): ZIO[R1, E1, (A, B)]
}
```

---

# Concurrency Operators

## Variants of ZipPar

---

# Concurrency Operators

## CollectAllPar and ForeachPar

```scala
object ZIO {
    def collectAllPar[R, E, A](
        in: Iterable[ZIO[R, E, A]]
    ): ZIO[R, E, List[A]]

    def foreachPar[R, E, A, B](
        in: Iterable[A]
    )(f: A => ZIO[R, E, B]): ZIO[R, E, List[B]]
}
```

---

# Concurrency Operators

## CollectAllParDiscard and ForeachParDiscard

```scala
object ZIO {
    def collectAllParDiscard[R, E, A](
        in: Iterable[ZIO[R, E, A]]
    ): ZIO[R, E, Unit]

    def foreachParDiscard[R, E, A, B](
        in: Iterable[A]
    )(f: A => ZIO[R, E, B]): ZIO[R, E, Unit]
}
```

---

# Concurrency Operators

## CollectAllParN and ForeachParN

```scala
object ZIO {
    def collectAllParN[R, E, A](
        n: Int
    )(in: Iterable[ZIO[R, E, A]]): ZIO[R, E, List[A]]

    def foreachParN[R, E, A, B](
        n: Int
    )(in: Iterable[A])(f: A => ZIO[R, E, B]): ZIO[R, E, List[B]]
}
```

---

# Concurrency Operators

## ValidatePar

```scala
object ZIO {
    def validatePar[R, E, A, B](
        in: Iterable[A]
    )(f: A => ZIO[R, E, B]): ZIO[R, ::[E], List[B]]
}
```

---

# Chapter 8: Fiber Supervison in Depth

![width:400px](/assets/missingno.png)

---

# Chapter 9: Interruption in Depth

---

# Timing

ZIO effects describe rather than do.

The ZIO runtime checks for interruption before executing each effect.


---

# Timing
## Before

```scala
for {
    fiber <- ZIO.succeed(println("Hello, World!")).fork
    _ <- fiber.interrupt
} yield ()
```

---

# Timing
## Before

```scala
for {
    ref <- Ref.make(false)
    fiber <- ZIO.never.ensuring(ref.set(true)).fork
    _ <- fiber.interrupt
    value <- ref.get
} yield value
```

---

# Timing
## Before

```scala
for {
    ref <- Ref.make(false)
    promise <- Promise.make[Nothing, Unit]
    fiber <- (promise.succeed(()) *> ZIO.never)
                .ensuring(ref.set(true))
                .fork
    _       <- promise.await
    _       <- fiber.interrupt
    value <- ref.get
} yield value
```

---

# Timing

## During

Anything inside a ZIO is considered a side effect and ZIO does not know how to interrupt it.

---

# Timing

## During

```scala
val effect: UIO[Unit] =
    UIO.succeed {
        var i = 0
        while (i < 100000) {
            println(i)
            i += 1
        }
    }

for {
    fiber <- effect.fork
    _ <- fiber.interrupt
} yield ()
```

---

# Timing

## attemptBlockingCancelable

```scala
object ZIO {
    def attemptBlockingCancelable(
        effect: => A
    )(cancel: UIO[Unit]): Task[A]
}
```

---

# Timing

```scala
import java.util.concurrent.atomic.AtomicBoolean

import zio.blocking._

def effect(canceled: AtomicBoolean): Task[Unit] =
    ZIO.attemptBlockingCancelable(
        {
            var i = 0
            while (i < 100000 && !canceled.get()) {
                println(i)
                i += 1
            }
        },
        UIO.succeed(canceled.set(true))
    )

for {
    ref <- ZIO.succeed(new AtomicBoolean(false))
    fiber <- effect(ref).fork
    _ <- fiber.interrupt
} yield ()
```
---

# Interruptible and Uninterruptible Regions

```scala
val uninterruptible: UIO[Unit] =
    UIO(println("Doing something really important"))
        .uninterruptible

val interruptible: UIO[Unit] =
    UIO(println("Feel free to interrupt me if you want"))
        .interruptible
```

---

# Interruptible and Uninterruptible Regions

⚠️ .uninterruptable is just another instruction ⚠️

```scala
for {
    ref <- Ref.make(false)
    fiber <- ref.set(true).uninterruptible.fork
    _ <- fiber.interrupt
    value <- ref.get
} yield value
```

---

# Interruptible and Uninterruptible Regions

Fixed

```scala
for {
    ref <- Ref.make(false)
    fiber <- ref.set(true).fork.uninterruptible
    _ <- fiber.interrupt
    value <- ref.get
} yield value
```

---

# Interruptible and Uninterruptible Regions

1. Combinators that change these setting apply to the entire scope they are invoked on.
2. Inner scopes override outer scopes.
3. Forked fibers inherit the settings of their parent fiber at the time they are forked.


---

# Interruptible and Uninterruptible Regions

```scala
(zio1 *> zio2 *> zio3).uninterruptible

(zio1 *> zio2.interruptible *> zio3).uninterruptible
```

---

# Composing Interruptibility

If you're working with your code all you need is interruptible and uninterruptible.

---

# Composing Interruptibility

What if you're writing operators for a ZIO effect? How do you make code interruptible or uninterruptible without changing the status of teh user's effect?

---

# Composing Interruptibility

## UninterruptibleMask

```scala
object ZIO {
    def uninterruptibleMask[R, E, A](
        k: ZIO.InterruptStatusRestore => ZIO[R, E, A]
    ): ZIO[R, E, A]
}
```

---

# Composing Interruptibility

## UninterruptibleMask

```scala
def myOperator[R, E, A](zio: ZIO[R, E, A]): ZIO[R, E, A] =
    ZIO.uninterruptibleMask { restore =>
        UIO(println("Some work that shouldn't be interrupted")) *>
        restore(zio) *>
        UIO(println("Some other work that shouldn't be interrupted"))
    }
```

---

# Composing Interruptibility

⚠️☢️☣️⚠️

Do not use interruptible or interruptibleMask.

---

# Composing Interruptibility

⚠️☢️☣️⚠️

```scala
import zio._
import zio.duration._

def delay[R, E, A](
    zio: ZIO[R, E, A]
)(duration: Duration): ZIO[R with Clock, E, A] =
    Clock.sleep(duration).interruptible *> zio
```

If Clock.sleep is interrupted, zio is never executed.

---

# Composing Interruptibility

⚠️☢️☣️⚠️

```scala
for {
    ref       <- ref.make(false)
    promise   <- Promise.make[Nothing, Unit]
    effect    = promise.succeed(()) *> ZIO.never
    finalizer = ref.set(true).delay(1.second)
    fiber     <- effect.ensuring(finalizer).fork
    _         <- promise.await
    _         <- fiber.interrupt
    value     <- ref.get
} yield value
```
---

# Composing Interruptibility

⚠️☢️☣️⚠️

Fix this by returning the exit code.

```scala
def saferDelay[R, E, A](zio: ZIO[R, E, A])(
    duration: Duration
): ZIO[R with Clock, E, A] =
    Clock.sleep(duration).interruptible.run *> zio
```

---

# Composing Interruptibility

⚠️☢️☣️⚠️

The interruptible combinator is only used four times in ZIO’s own source code. Do not use it.

---

# Waiting for Interruption

---

# Waiting for Interruption

This blocks for 5 seconds.

```scala
for {
    promise   <- Promise.make[Nothing, Unit]
    effect    = promise.succeed(()) *> ZIO.never
    finalizer = ZIO.succeed(println("Closing file"))
                 .delay(5.seconds)
    fiber     <- effect.ensuring(finalizer).fork
    _         <- promise.await
    _         <- fiber.interrupt
    _         <- ZIO.succeed(println("Done interrupting"))
} yield ()
```

---

# Waiting for Interruption

This forks the interrupt and executes the finalizer on a new Fiber.

```scala
for {
    promise   <- Promise.make[Nothing, Unit]
    effect    = promise.succeed(()) *> ZIO.never
    finalizer = ZIO.succeed(println("Closing file"))
                 .delay(5.seconds)
    fiber     <- effect.ensuring(finalizer).fork
    _         <- promise.await
    _         <- fiber.interrupt.fork
    _         <- ZIO.succeed(println("Done interrupting"))
} yield ()
```

---

# Waiting for Interruption

This disconnects the ZIO effect from the interrupt and executes the finalizer on a new Fiber.

```scala
for {
    promise   <- Promise.make[Nothing, Unit]
    effect    = promise.succeed(()) *> ZIO.never
    finalizer = ZIO.succeed(println("Closing file"))
                 .delay(5.seconds)
                 .disconnect
    fiber     <- effect.ensuring(finalizer).fork
    _         <- promise.await
    _         <- fiber.interrupt.fork
    _         <- ZIO.succeed(println("Done interrupting"))
} yield ()
```

---

# Questions?
