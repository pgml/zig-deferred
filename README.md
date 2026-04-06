> [!NOTE]
> #### This project is hosted and maintained on Codeberg:
> #### https://codeberg.org/pgml/zig-deferred
> 

# zig-deferred

### A tiny single-threaded execution queue with built-in debouncing.


## Installation

```sh
zig fetch --save git+https://codeberg.org/pgml/zig-deferred.git
```

Then add the following to `build.zig`:

```zig
const deferred = b.dependency("zig_deferred", .{});
exe.root_module.addImport("zig_deferred", deferred.module("zig_deferred"));
```

## API
```zig
const FnCtx = struct {
    func: *const fn (*anyopaque) void,
    ctx: *anyopaque,
};

pub const Event = struct {
    /// Key for debouncing.
    key: ?[]const u8 = null,

    /// Timestamp in ms when this event should be executed.
    due: i64,

    /// Main callback.
    cb: FnCtx,

    /// Optional callback after `cb`.
    onAfterExecution: ?FnCtx = null,
};


/// Extends the queue by 1 event.
pub fn append(self: *Queue, event: Event) !void;

/// Inserts or replaces an event in the queue.
///
/// If an event with the same `key` already exists, it is overwritten.
/// Otherwise, the event is appended.
pub fn put(self: *Queue, event: Event) !void;

/// Returns the event at `index` or null.
///
/// Asserts that the list is not empty.
/// Asserts that the index is in bounds.
pub fn get(self: Queue, index: usize) ?Event;

/// Returns whether an event with `key` is in the queue.
pub fn contains(self: Queue, key: []const u8) bool;

/// Removes an event at `index` and returns it.
pub fn removeAt(self: *Queue, index: usize) Event;

/// Removes an event by a `key` and returns the event or null
/// if no event is found.
pub fn remove(self: *Queue, key: []const u8) ?Event;

/// The amount of scheduled events.
pub fn len(self: Queue) usize;
```

## Usage
```zig
const std = @import("std");
const deferred = @import("zig_deferred");

const App = struct {
    counter: u16 = 0,
    event_queue: ?deferred.Queue = null,

    pub fn count(self: *App) void {
        self.counter += 1;
    }
};

fn foo(ctx: *anyopaque) void {
    const app: *App = @ptrCast(@alignCast(ctx));
    app.count();

    std.debug.print("({}) All your {s} are belong to us.\n", .{
        app.counter,
        "codebase",
    });
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer if (gpa.deinit() == .leak) std.process.exit(1);
    const alloc = gpa.allocator();

    var app: App = .{};
    app.event_queue = .init(alloc);

    if (app.event_queue) |*queue| {
        try queue.append(.{
            .due = 1000,
            .cb = .{ .func = foo, .ctx = &app },
        });

        try queue.append(.{
            .due = 2000,
            .cb = .{ .func = foo, .ctx = &app },
        });

        // debouncing example
        for (0..5) |_| {
            try queue.put(.{
                .key = "debounce",
                .due = 1500,
                .cb = .{ .func = foo, .ctx = &app },
            });
        }
    }

    while (true) {
        if (app.event_queue) |*queue| {
            try queue.run();
        }
    }

    // just a safety measure. Usually not required.
    if (app.event_queue) |*queue| {
        queue.deinit();
    }
}
```
