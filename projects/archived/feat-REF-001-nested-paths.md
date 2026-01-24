# feat-REF-001: Variable Verbs Take References for Nested Collections

## Priority
**High** - Enables natural nested data manipulation

## Problem
Current `/set`, `/get`, `/drop` take variable names as strings with single-level indexing:
```
/set "varName", *value;          :: First param is string
/set "list", 0, *value;          :: Can set list[0]
/set "map", "key", *value;       :: Can set map["key"]
```

But nested access is impossible:
```
/set "map", "nested", "deep", *value;  :: Doesn't work!
```

Users cannot access: `*data["users"][0]["name"]`

## Proposed Change
References carry a path of indexes, enabling nested access:
```
*data["users"][0]["name"] <- "Alice";
<- *data["config"]["server"]["port"];
```

## Reference AST Change

### Current
```csharp
public record ReferenceValue(
    string Name,
    IValue? Index  // Single index
);
```

### Proposed
```csharp
public record ReferenceValue(
    string Name,
    IReadOnlyList<IValue> Path  // Multiple indexes
);
```

## Parsing Changes

### Reference Syntax
```
*var                    → ReferenceValue("var", [])
*var[0]                 → ReferenceValue("var", [0])
*var["key"]             → ReferenceValue("var", ["key"])
*var[0]["key"]          → ReferenceValue("var", [0, "key"])
*var["a"][0]["b"]       → ReferenceValue("var", ["a", 0, "b"])
```

### Parser Update
```csharp
ReferenceValue ParseReference()
{
    Consume('*');
    var name = ParseIdentifier();
    var path = new List<IValue>();

    while (Peek() == '[')
    {
        Consume('[');
        var index = ParseExpression();
        Consume(']');
        path.Add(index);
    }

    return new ReferenceValue(name, path);
}
```

## Verb Changes

### /set
```
/set *var[path...], value;
```

The reference carries its path; value is the last parameter.

```csharp
public class SetDriver : IVerbDriver
{
    public VerbResult Execute(IExecutionContext context, VerbCallAst call)
    {
        if (call.UnnamedParams.Count < 2)
            return Fatal("parameter_not_found", "Set requires variable and value");

        var target = call.UnnamedParams[0];
        var value = call.UnnamedParams[^1]; // Last param is value

        if (target is ReferenceValue refVal)
        {
            return SetAtPath(context, refVal.Name, refVal.Path, value);
        }
        else if (target is StringValue strVal)
        {
            // Legacy: string name with explicit indexes
            var name = strVal.Value;
            var path = call.UnnamedParams.Skip(1).Take(call.UnnamedParams.Count - 2).ToList();
            return SetAtPath(context, name, path, value);
        }

        return Fatal("invalid_type", "Expected reference or string");
    }

    private VerbResult SetAtPath(IExecutionContext context, string name, IReadOnlyList<IValue> path, IValue value)
    {
        if (path.Count == 0)
        {
            context.SetVariable(name, value);
            return VerbResult.Ok();
        }

        var current = context.GetVariable(name);
        if (current is NothingValue)
            return Fatal("invalid_path", $"Variable '{name}' does not exist");

        // Navigate to parent of final element
        for (int i = 0; i < path.Count - 1; i++)
        {
            current = GetIndex(current, path[i]);
            if (current is NothingValue)
                return Fatal("invalid_path", $"Path element {i} does not exist");
        }

        // Set final element
        var finalIndex = path[^1];
        return SetIndex(current, finalIndex, value);
    }
}
```

### /get
```
/get *var[path...]; -> *result
<- *var[path...];
```

```csharp
public class GetDriver : IVerbDriver
{
    public VerbResult Execute(IExecutionContext context, VerbCallAst call)
    {
        var target = call.UnnamedParams[0];

        IValue result;
        if (target is ReferenceValue refVal)
        {
            result = GetAtPath(context, refVal.Name, refVal.Path);
        }
        else if (target is StringValue strVal)
        {
            var name = strVal.Value;
            var path = call.UnnamedParams.Skip(1).ToList();
            result = GetAtPath(context, name, path);
        }
        else
        {
            return Fatal("invalid_type", "Expected reference or string");
        }

        return VerbResult.Ok(result);
    }

    private IValue GetAtPath(IExecutionContext context, string name, IReadOnlyList<IValue> path)
    {
        var current = context.GetVariable(name);

        foreach (var index in path)
        {
            current = GetIndex(current, index);
            if (current is NothingValue)
                return NothingValue.Instance;
        }

        return current;
    }
}
```

### /drop
```
/drop *var[path...];
```

For nested paths, sets the element to `?` (nothing). For root variables (empty path), removes the variable entirely.

## Syntactic Sugar Update

The assignment sugar already uses references:
```
*data["users"][0]["name"] <- "Alice";
:: Desugars to: /set *data["users"][0]["name"], "Alice";

<- *data["users"][0]["name"];
:: Desugars to: /get *data["users"][0]["name"];
```

The reference now carries the full path, so sugar works naturally.

## Helper Functions

```csharp
static class CollectionHelpers
{
    public static IValue GetIndex(IValue collection, IValue index)
    {
        return (collection, index) switch
        {
            (ListValue list, IntegerValue i) =>
                i.Value >= 0 && i.Value < list.Count ? list[(int)i.Value] : NothingValue.Instance,
            (MapValue map, StringValue key) =>
                map.TryGetValue(key.Value, out var val) ? val : NothingValue.Instance,
            (MapValue map, IntegerValue key) =>
                map.TryGetValue(key.Value.ToString(), out var val) ? val : NothingValue.Instance,
            _ => NothingValue.Instance
        };
    }

    public static VerbResult SetIndex(IValue collection, IValue index, IValue value)
    {
        switch (collection, index)
        {
            case (ListValue list, IntegerValue i):
                if (i.Value < 0 || i.Value >= list.Count)
                    return VerbResult.Fatal("index_out_of_range", ...);
                list[(int)i.Value] = value;
                return VerbResult.Ok();

            case (MapValue map, StringValue key):
                map[key.Value] = value;
                return VerbResult.Ok();

            case (MapValue map, IntegerValue key):
                map[key.Value.ToString()] = value;
                return VerbResult.Ok();

            default:
                return VerbResult.Fatal("invalid_index_operation", ...);
        }
    }
}
```

## Spec Update

Add to spec.md in References section:

```markdown
### Reference Paths

References can include multiple index accessors for nested collection access:

    *var[index1][index2][index3]...

Each index is evaluated and used to navigate into nested collections:
- For lists: integer index (0-based)
- For maps: string or integer key

Examples:
    *users[0]                     :: First element of users list
    *config["server"]             :: "server" key of config map
    *data["users"][0]["name"]     :: Nested: data.users[0].name

Assignment sugar works with paths:
    *data["users"][0]["name"] <- "Alice";
    <- *data["config"]["port"];

Path navigation stops at `?` (nothing):
    *data["missing"]["key"]       :: Returns ? if "missing" doesn't exist
```

## Test Cases

### Parser Tests
```csharp
[Fact]
public void Reference_WithSingleIndex_ParsesPath()
{
    var expr = Parse("*list[0]");
    var refVal = Assert.IsType<ReferenceValue>(expr);
    Assert.Equal("list", refVal.Name);
    Assert.Single(refVal.Path);
    Assert.Equal(0, ((IntegerValue)refVal.Path[0]).Value);
}

[Fact]
public void Reference_WithMultipleIndexes_ParsesPath()
{
    var expr = Parse("*data[\"users\"][0][\"name\"]");
    var refVal = Assert.IsType<ReferenceValue>(expr);
    Assert.Equal("data", refVal.Name);
    Assert.Equal(3, refVal.Path.Count);
    Assert.Equal("users", ((StringValue)refVal.Path[0]).Value);
    Assert.Equal(0, ((IntegerValue)refVal.Path[1]).Value);
    Assert.Equal("name", ((StringValue)refVal.Path[2]).Value);
}

[Fact]
public void Reference_WithNoIndex_HasEmptyPath()
{
    var expr = Parse("*simple");
    var refVal = Assert.IsType<ReferenceValue>(expr);
    Assert.Empty(refVal.Path);
}
```

### Runtime Tests
```csharp
[Fact]
public void Set_NestedMapValue_Works()
{
    var story = Parse(@"
Test
===
/set *data, {""user"": {""name"": ""Bob""}};
*data[""user""][""name""] <- ""Alice"";
");
    var context = Execute(story);
    var data = (MapValue)context.GetVariable("data");
    var user = (MapValue)data["user"];
    Assert.Equal("Alice", ((StringValue)user["name"]).Value);
}

[Fact]
public void Get_NestedListValue_Works()
{
    var story = Parse(@"
Test
===
/set *data, [[1, 2], [3, 4]];
<- *data[1][0]; -> *result
");
    var context = Execute(story);
    Assert.Equal(3, ((IntegerValue)context.GetVariable("result")).Value);
}

[Fact]
public void Set_MissingPath_ReturnsFatal()
{
    var story = Parse(@"
Test
===
/set *data, {};
*data[""missing""][""key""] <- ""value"";
");
    var result = Execute(story);
    Assert.True(result.HasFatal("invalid_path"));
}

[Fact]
public void Get_MissingPath_ReturnsNothing()
{
    var story = Parse(@"
Test
===
/set *data, {};
<- *data[""missing""][""key""]; -> *result
");
    var context = Execute(story);
    Assert.IsType<NothingValue>(context.GetVariable("result"));
}
```

## Backward Compatibility
- Single-index paths still work: `*var[0]` becomes `ReferenceValue("var", [0])`
- Old code continues to work unchanged
- String-based `/set "name", index, value` syntax still supported

## Verification
```bash
cd c#
dotnet test --filter "Reference"
dotnet test --filter "Set"
dotnet test --filter "Get"
```

## Files to Modify
- `c#/src/Zoh.Compiler/Ast/Values/ReferenceValue.cs` - Change Index to Path
- `c#/src/Zoh.Compiler/Parser/Parser.cs` - Parse multiple indexes
- `c#/src/Zoh.Runtime/Verbs/Var/SetDriver.cs` - Handle paths
- `c#/src/Zoh.Runtime/Verbs/Var/GetDriver.cs` - Handle paths
- `c#/src/Zoh.Runtime/Verbs/Var/DropDriver.cs` - Handle paths
- `c#/src/Zoh.Runtime/Helpers/CollectionHelpers.cs` - Add path navigation
- `c#/tests/Zoh.Tests/Parser/ReferenceParsingTests.cs` - Add tests
- `c#/tests/Zoh.Tests/Runtime/NestedAccessTests.cs` - Add tests
- `spec.md` - Document reference paths
