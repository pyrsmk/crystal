Union-typed method call result is not snapshot before evaluating the next operand in multidispatch

When comparing two method calls where the left operand returns a union type (`Int32?`) and the right operand has a side effect that mutates the same underlying memory, the comparison always evaluates to `true` — even though the values should differ.

### Reproduction

```crystal
class Inner
  @pointer : Int32?
  @size : Int32

  def initialize(@size : Int32, @pointer : Int32? = nil); end

  def pointer : Int32?
    @pointer
  end

  def pointer=(index : Int32) : Nil
    @pointer = index
  end

  def next : Int32?
    return nil if @pointer.nil?

    @pointer = @pointer.not_nil! + 1
    @pointer = @size - 1 if @pointer.not_nil! > @size - 1
    @pointer
  end
end

class Wrapper
  def initialize(@set : Inner); end

  def next_buggy : Int32?
    return nil if @set.pointer.nil?
    @set.pointer = 0 if @set.pointer == @set.next
    @set.pointer
  end

  def next_fixed : Int32?
    return nil if @set.pointer.nil?
    a = @set.pointer
    b = @set.next
    @set.pointer = 0 if a == b
    @set.pointer
  end
end

puts "=== Buggy ==="
wrapper = Wrapper.new(Inner.new(2, 0))
3.times { |i| puts "call #{i}: pointer=#{wrapper.next_buggy}" }

puts "\n=== Fixed ==="
wrapper2 = Wrapper.new(Inner.new(2, 0))
3.times { |i| puts "call #{i}: pointer=#{wrapper2.next_fixed}" }
```

### Expected output

Both versions should produce the same result:

```
call 0: pointer=1
call 1: pointer=0
call 2: pointer=1
```

### Actual output

```
=== Buggy ===
call 0: pointer=0
call 1: pointer=0
call 2: pointer=0

=== Fixed ===
call 0: pointer=1
call 1: pointer=0
call 2: pointer=1
```

The buggy version always returns `0` — the pointer never advances.

### Root cause

The issue is in `codegen_dispatch` (`src/compiler/crystal/codegen/call.cr`).

When Crystal evaluates `@set.pointer == @set.next`, both sides return `Int32?` (a `MixedUnionType`, which is `passed_by_value?`). The codegen proceeds in two phases:

1. **Evaluate operands and extract type IDs** — `request_value(node_obj)` evaluates `@set.pointer` and stores the result in `%self`. For `passed_by_value?` types, `@last` is a **pointer to the union struct inside the original object's memory** (the `@pointer` field of `Inner`). Then `request_value(arg)` evaluates `@set.next`, which **mutates that same `@pointer` field** as a side effect.

2. **Compare actual values in `current_def` block** — Crystal re-reads the value from `%self`, which still points to the original field. Since `@set.next` already modified it, both sides now hold the same value. The comparison always returns `true`.

This can be confirmed in the LLVM IR: the type ID of the left operand is loaded **before** the call to `next`, but the actual integer value is loaded **after** — from the same memory address that `next` already mutated.

### Suggested fix

In `codegen_dispatch`, after evaluating each operand via `request_value`, copy `passed_by_value?` results to a stack-allocated temporary (`alloca` + `assign`) before evaluating the next operand. This snapshots the value so that subsequent evaluations cannot corrupt it.
