---
layout:     post
title:      "Design Patterns on Competitive Programming"
date:       2022-10-06 03:00:00 +0700
categories: code
---

# Introduction

It's rare sight to see Design Patterns being used on Competitive Programming type of problem.
However, after working
on [uVA - 1721 - Window Manager](https://onlinejudge.org/index.php?option=com_onlinejudge&Itemid=8&category=24&page=show_problem&problem=4794)
for hours, I found myself lost on a thousand lines of codes that doesn't make sense with tons of bugs.

The problem is easy to understand.
You're asked to simulate how operating system manage their windows.
There are 4 set of operations, **OPEN** new window, **CLOSE** a window, **RESIZE** a window, and **MOVE** a window (and
the collided windows).

To my surprise, applying design patterns into my solution helps make the code much more sensible.
I combined [Prototype](https://refactoring.guru/design-patterns/prototype)
and [Memento](https://refactoring.guru/design-patterns/memento)
patterns to build my own patterns: `Object Transaction Manager` and `Transactional Object`.
This patterns enables me to check the effect of certain operation and make a before-after comparison.

The full accepted solution can be seen
on [dadangeuy/uhunt](https://github.com/dadangeuy/uhunt/blob/main/src/main/java/dev/rizaldi/uhunt/c1/p1721/Main.java).

# Design Patterns

## Object Transaction Manager

This pattern was inspired by transaction feature in databases.
It's a simplified version of [Memento](https://refactoring.guru/design-patterns/memento) pattern.

Client should be able to create snapshot of an object at any point in time, and use it to restore the object's state.
Object Transaction Manager handle the lifecycle of the snapshot.

```java
final class Transaction<T extends Snapshot<T>> {
    private final T object;
    private T snapshot;

    public Transaction(T object) {
        this.object = object;
        this.snapshot = null;
    }

    public void start() {
        if (snapshot == null) snapshot = object.snapshot();
    }

    public void commit() {
        snapshot = null;
    }

    public void rollback() {
        if (committed()) return;
        object.restore(snapshot);
        commit();
    }

    public T read(boolean committed) {
        return !committed || committed() ? object : snapshot;
    }

    public boolean committed() {
        return snapshot == null;
    }
}
```

## Transactional Object

Transactional Object means the object's snapshot lifecycle is managed by `Transaction Manager`.
In our case, `Window` is a Transactional Object because `Transaction` has access to create and restore its snapshot.

```java
interface Snapshot<T> {
    T snapshot();

    void restore(T snapshot);
}
```

```java
final class Window extends ImmutableWindow implements Snapshot<Window> {
    public final Transaction<Window> transaction;

    public Window(int x, int y, int width, int height) {
        super(x, y, width, height);
        this.transaction = new Transaction<>(this);
    }

    @Override
    public Window snapshot() {
        return new Window(x(), y(), width(), height());
    }
    
    @Override
    public void restore(Window snapshot) {
        x = snapshot.x();
        y = snapshot.y();
        width = snapshot.width();
        height = snapshot.height();
    }
}
```

# Algorithms

## Resize

Resize algorithm was simple:

- Find window located in (x, y)
- Change their width and height.
- If it goes beyond the screen OR overlap with other windows, rollback.

```java
final class Screen {
    public void resize(int x, int y, int w, int h) throws InternalError {
        Window resized = target(x, y);
        resized.transaction.start();
        resized.resize(w, h);
        if (outside(resized) || overlap(resized)) {
            resized.transaction.rollback();
            throw new UnfitWindowError();
        }
        resized.transaction.commit();
    }
}
```

## Move

Move was complicated.
We need to move a window and the collided windows, thus I decide to use recursive strategy to push the windows.

```java
final class Screen {
    private void forcePush(Window moved, int dx, int dy) {
        Window movement = moved.snapshot().stretch(dx, dy);

        // push collided windows
        for (Window existing : windows) {
            boolean collided = existing != moved && existing.overlap(movement);
            if (!collided) continue;

            int leftDx = movement.left() - existing.right(), rightDx = movement.right() - existing.left();
            int topDy = movement.top() - existing.bottom(), bottomDy = movement.bottom() - existing.top();
            int newDx = dx < 0 ? leftDx : dx > 0 ? rightDx : 0;
            int newDy = dy < 0 ? topDy : dy > 0 ? bottomDy : 0;
            forcePush(existing, newDx, newDy);
        }

        // push window
        moved.transaction.start();
        moved.move(dx, dy);
    }
}
```

Next, the windows might get pushed out of the screen, thus we need to pull it back into the screen.

```java
final class Screen {
    private void pullToScreen(List<MutableWindow> targets, int dx, int dy) {
        int minLeft = targets.stream().mapToInt(Window::left).min().orElse(0);
        int maxRight = targets.stream().mapToInt(Window::right).max().orElse(0);
        int minTop = targets.stream().mapToInt(Window::top).min().orElse(0);
        int maxBottom = targets.stream().mapToInt(Window::bottom).max().orElse(0);
        int pullDx = minLeft < 0 ? -minLeft : maxRight > width ? width - maxRight : 0;
        int pullDy = minTop < 0 ? -minTop : maxBottom > height ? height - maxBottom : 0;
        targets.forEach(t -> t.move(pullDx, pullDy));
        // ...
    }
}
```

Be careful, some windows might get pulled too far and ended up going to the opposite direction. In such cases, we can
infer that the window doesn't collide with the other windows, thus we can revert it to their original position.

```java
final class Screen {
    private void pullToScreen(List<MutableWindow> targets, int dx, int dy) {
        // ...
        Direction direction = Direction.of(dx, dy);
        targets.stream().filter(t -> !direction.equals(t.direction())).forEach(t -> t.transaction.rollback());
    }
}
```

Finally, combine the logic above to compose the move logic.
Push the windows to the outside of the screen, then pull it back to inside the screen.

```java
final class Screen {
    public void move(int x, int y, int dx, int dy) throws InternalError {
        Window moved = target(x, y);

        forcePush(moved, dx, dy);
        pullToScreen(changes(), dx, dy);

        int actualDx = moved.dx(), actualDy = moved.dy();
        windows.forEach(w -> w.transaction.commit());

        if (actualDx != dx) throw new UnexpectedMoveError(dx, actualDx);
        if (actualDy != dy) throw new UnexpectedMoveError(dy, actualDy);
    }
}
```
