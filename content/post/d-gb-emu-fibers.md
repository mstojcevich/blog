---
title: "Using D fibers to implement the Gameboy's timing"
date: 2018-11-05T22:58:41-05:00
draft: false
---

Hi there! It's been over a year and a half since I last wrote a post about my Gameboy emulator. This time I want to go over an idea I had for a unique-ish strategy of dealing with timing in video game emulators. This is applying the concept of [coroutines](https://en.wikipedia.org/wiki/Coroutine) (1958) to a Gameboy emulator.

## An overview of emulation timing

In the hardware of game systems such as the Gameboy and Nintendo Entertainment System, there are many different components operating at the same time. For example, the CPU executes instructions during the same time that the pixel processor (PPU) draws sprites to the screen. 

In real hardware, these separate processors are clocked using a shared base clock, meaning that they perform these operations in a way which is predictable by the other components. For example, the CPU can predict exactly when it should write to video memory to ensure that its changes get picked up for the current line. Games can use this knowledge to perform certain tricks, like having the CPU change pixel data in the middle of a scanline render to create certain visual effects.

In the example of the Gameboy's PPU and CPU, both processors can perform certain complex operations which span multiple clock cycles. For the Gameboy, an example of this is the `PUSH BC` instruction. This instruction pushes two bytes onto the stack, from registers `B` and `C` respectively. Because the Gameboy takes one clock cycle to write a single byte, this operation will span two clock cycles. If the stack pointer was inside of video memory, and the PPU happened to read these two bytes while `PUSH BC` was in the middle of running, it could behave differently than if both bytes were written at once.

Below is a simplified (not necessarily accurate) illustration of the problem:

![Timing Example Diagram](/timing.svg)



## A solution to the problem
In my Gameboy emulator, I currently have an inaccurate but functioning solution to the `POP BC` problem mentioned above. Whenever the CPU needs to read from memory, it will simulate one tick's worth of work for the other components.

There is a problem with this solution: when the CPU tries to simulate the PPU for one tick, the PPU often cannot do anything meaningful in just one cycle. The PPU can't just call the CPU's update after doing one tick's worth of work ([mutual recursion](https://en.wikipedia.org/wiki/Mutual_recursion)), since we'd end up running out of room in the call stack as this will clearly recurse indefinitely.

To solve this problem, my current PPU implementation will accumulate cycles until enough time has passed for the PPU to do something meaningful; at this point, the PPU will do all of the operations at once. This solution is problematic. If the PPU performs every operation at once, there are cases where it will read data that is "too new" and act in a way that is unfaithful to the original system. For example, after enough cycles have gone by to render one scanline, my emulator will then perform all of the actions required to render that scanline. Most games do not attempt to be tricky enough to change a scanline while it's being rendered, but there are games and tech demos out there that do.

Ok then, what's the solution? To get at a better solution, we must consider our ideal end goal. Ideally we would have a way to suspend execution of one component, have it bookmark where it was, run every other component until they respectively suspend, then resume from where we left off. Implementing this is where D's fiber support comes in.

## Fibers

Fibers (similar to coroutines, goroutines, deferreds, etc.) are typically used as a way to abstract concurrent computing through "cooperative multitasking". As opposed to having the operating system switch the current thread after a certain amount of time, fibers will typically indicate that they're ready to be switched away by `yield`ing execution to another fiber. Fibers can be implemented entirely in userspace by sharing a single OS thread or be split up to run in parallel. These are super useful when trying to have a lightweight non-blocking single threaded application; as a result, fibers are a common concept applied in server frameworks such as Twisted, Node.js, and Golang.

The D programming language implements first-class support for fibers. Many languages which provide support for fibers or coroutines (such as Go) will include a built-in fiber scheduler which will provide an intelligent mapping between fibers and OS threads, allowing for parallelism. Interestingly, D doesn't constrain us to a specific built-in fiber scheduler. In D, a program is able to call fibers directly, and check their status once they yield. Here's an example adapted from [D's excellent language documentation](https://dlang.org/phobos/core_thread.html#.Fiber):
```D
void fiberFunc()
{
    counter += 4;
    Fiber.yield();
    counter += 8;
}

void fiberFunc2() {
    counter += 2;
}

// create instances of each type
Fiber fiber1 = new Fiber( &fiberFunc );
Fiber fiber2 = new Fiber( &fiberFunc2 );

assert( counter == 0 );

fiber2.call(); // Should add 2
assert( counter == 2, "Derived fiber increment." );

fiber1.call(); // Should add 4 first
assert( counter == 6, "First composed fiber increment." );

counter += 16;
assert( counter == 22, "Calling context increment." );

fiber1.call(); // Should add 8 now
assert( counter == 30, "Second composed fiber increment." );

// since each fiber has run to completion, each should have state TERM
assert( fiber1.state == Fiber.State.TERM );
assert( fiber2.state == Fiber.State.TERM );
```

We're going to abuse D's fiber support by providing a non-parallel entirely-fair implementation of fiber scheduling, which will simulate the behavior we desire for hardware component timing. We will transform D's fibers into something closer to what is typically referred to as a coroutine, essentially a fiber without complex scheduling or parallelism.


## Our fiber scheduler

As mentioned above, we want to implement a fiber scheduler that properly interleaves execution of different hardware components; thankfully, D makes this incredibly easy. Eventually we may want to have this scheduler implementation serve as an arbiter for cross-component communication and ensure that data is only written on the falling clock edge, but for now this is sufficient.

```D
// Do similar implementation for CPU
class PPU extends Fiber {
    this()
    {
        super( &run );
    }

private :
    void run()
    {
        while (true) {
            // do stuff that would take one cycle
            Fiber.yield();
            // do more stuff that would take another cycle
            Fiber.yield();
            // ...
            Fiber.yield();
        }
    }
}

class ComponentScheduler {

    private CPU cpu;
    private PPU ppu;

    this(CPU cpu, PPU ppu) {
        this.cpu = cpu;
        this.ppu = ppu;
    }

    void run() {
        while (true) {
            this.cpu.call();
            this.ppu.call();
        }
    }
}
```

This is very neat and should help to write a clean and easy-to-understand emulator. It should be straightforward to extend this to other components such as the timer and the audio subsystem.

