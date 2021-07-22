<style>
  h2 {padding-top: 30px}
</style>

# Q-SYS Lua Design Patterns

This is a collection of *[design patterns](https://en.wikipedia.org/wiki/Software_design_pattern)* (and some *[anti-patterns](https://en.wikipedia.org/wiki/Anti-pattern)*) for writing Lua code in Q-SYS.

This may also stray from strictly covering design patterns and into general tips / etc, but I trust you won't mind!


## Control / EventHandler

The obvious way to attach some logic to a Q-SYS control is using an *event handler*. The attached function will run whenever the control changes:

```lua
Controls.MyButton.EventHandler = function(c)
  if(c.Boolean) then
    c.Color = 'red'
  else
    c.Color = 'green'
  end
end
```

> **Note:** This example could be made more succinct using the *[Ternary](Ternary)* pattern.

The problem with this is that it doesn't run at start-up - so the following pattern exists for ensuring the logic is always consistent:

1. Define a function to execute the logic.
2. Attach this function as an Event Handler.
3. Run this function at start-up.

### Example

```lua
-- Define function
function MyButtonHandler(c)
  if(c.Boolean) then
    c.Color = 'red'
  else
    c.Color = 'green'
  end
end

-- Attach event handler
Controls.MyButton.EventHandler = MyButtonHandler

-- Run the function now
MyButtonHandler(Controls.MyButton)
```

## Ternary

Sometimes, as in the example above, we wish to turn a boolean value (such as the state of a button) into two different values - one for true, and one for false.

In the example above, we convert `Controls.MyButton.Boolean` into `'red'` or `'white'`.

As long as the value we wish to use for `true` is not `false` or `nil`, we can exploit Lua's `and` and `or` operators to achieve this in a more succinct way, such that:

```lua
if(c.Boolean) then
  c.Color = 'red'
else
  c.Color = 'green'
end
```
becomes:
```lua
c.Color = c.Boolean and 'red' or 'green'
```

## Conjecture / counterexample

Often, we wish to find the best candidate from a set, or establish whether some condition is true for all members of a set. A simple logical pattern for achieving this is to:

1. Make an assumption about the result that can be disproved by an individual member.
2. Test the assumption against each member, and amend the assumption accordingly.

### Check condition

For example, if we wish to check that all of the elements in a table array are true:
```lua
MyTableArray = { true, true, true, false, true }
```
then we can start by assuming they are all true, and then check each element to see if it is false:
```lua
-- assume all members are false
local allAreTrue = true

-- for each member
for _,e in pairs(MyTableArray) do

  -- if it is false (not true)
  if(not e) then

    -- our assumption is false
    allAreTrue = false

  end
end
```

## Best candidate

We can also use this pattern to find the best candidate from a set, if we make a new assumption when our old one is disproven. For example, if we wish to find the index of the highest number in a table array:

```lua
MyTableArray = { 9, 3, 4, 6, 7, 12, 3, 7 }
```
then we could start by assuming that the first member is the highest number, and then check each of the remaining numbers in turn to see if they are higher:
```lua
-- assume the first element is the best (highest)
local bestTableIndex = 1
local highestNumber = MyTableArray[1]

for index, number in pairs(MyTableArray) do

  -- if this number is higher than the highest we've seen so far
  if(number > highestNumber) then

    -- make a new assumption that this is the best (highest) number
    bestTableIndex = index
    highestNumber = number

  end
end

print(bestTableIndex, highestNumber)
-- >  6               12
```

## If true, set true - **anti-pattern**

Instead of:

```lua
if(Controls.MyButton.Boolean) then
  Controls.MyLED.Boolean = true
else
  Controls.MyLED.Boolean = false
end
```
write:
```lua
Controls.MyLED.Boolean = Controls.MyButton.Boolean
```

## TCP Socket Template

This is a standard boilerplate for creating TCP connections in Q-SYS Lua that provides several advantages over the example given in the help file:

1. It's shorter.
2. Handling of socket events and socket data are treated as two concerns.

```lua
-- Create a new socket
Sock = TcpSocket.New()
```

TCP socket events are typically matched against the `TcpSocket.Events` table - however, the event itself is just a string, such as `"CONNECTED"`.

Rather than look up each socket event individually, we can just print that underlying string:

```lua
-- Whenever the socket status changes, print the new status
Sock.EventHandler = print
-- > userdata: 00000250FF9FB928   CONNECTED	
```
The only downside of this is that it also attempts to print the socket itself as a string (this is the "userdata" we see). For debugging, this is likely sufficient. For a final plugin or reusable component, we may wish to link this to a *Status* control instead - assuming we have a status indicator control called `Status`:

```lua
Sock.EventHandler = function(_, evt)
  if(evt == TcpSocket.Events.Connected) then
    Controls.Status.Value = 0
  elseif(evt == TcpSocket.Events.Reconnect) then
    Controls.Status.Value = 5
  else
    Controls.Status.Value = 2
  end
  Controls.Status.String = evt
end
```
We can then handle data reception events separately:

```lua
Sock.Data = function()
  -- process data here
  -- see "Line-by-line socket data"
  -- or "Fixed-length socket data"
end
```

And finally we can set up the socket connection. Assuming we have a text control for the device IP / hostname, we can attach this using the *[Control / EventHandler](#control-eventhandler)* pattern:

```lua
function reconnect()
  if(Sock.IsConnected) then Sock:Disconnect() end
  if(Controls.IP.String ~= '') then
    Sock:Connect(Controls.IP.String, 80)
  end
end
Controls.IP.EventHandler = reconnect
reconnect()
```

## Line-by-line socket data

The example given in the help file for reading socket data line-by-line is:
```lua
print( "socket has data" )
message = sock:ReadLine(TcpSocket.EOL.Any)
while (message ~= nil) do
  print( "reading until CrLf got "..message )
  message = sock:ReadLine(TcpSocket.EOL.Any)
end
```
However, this involves repetition of the `message = sock:ReadLine...` line.

We can avoid this by defining an anonymous function for a `for ... in` loop to call. (See ["generic for"](https://www.lua.org/pil/4.3.5.html) in the Lua Reference Manual).

```lua
print( "socket has data" )
local function read()
  return Sock:ReadLine(TcpSocket.EOL.Any)
end
for message in read do
  print( "reading until CrLf got " .. message)
end
```

Aside from the deduplication, this pattern also has the advantage that the logic inside the `read()` function can be made more complex for protocols that require it.

