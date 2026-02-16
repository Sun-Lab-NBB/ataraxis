# Class and design patterns

Conventions for C# classes, MonoBehaviour components, ScriptableObjects, enums, properties,
inheritance, and Unity-specific patterns in Sun Lab projects.

---

## MonoBehaviour patterns

MonoBehaviour is the base class for Unity components attached to GameObjects. Most Sun Lab C#
classes inherit from MonoBehaviour.

### Lifecycle method ordering

Unity lifecycle methods must appear in their natural execution order:

```csharp
/// <summary>Manages an infinite corridor VR task with probabilistic segment transitions.</summary>
public class Task : MonoBehaviour
{
    // Constants
    // Public fields (Inspector-serialized)
    // Private fields

    /// <summary>Called once before Start. Initializes references and validates configuration.</summary>
    void Awake()
    {
        // Early initialization, component references
    }

    /// <summary>Called once after Awake. Performs setup that depends on other components.</summary>
    void Start()
    {
        // Setup that depends on other components being initialized
    }

    /// <summary>Called every frame. Updates task state and checks for transitions.</summary>
    void Update()
    {
        // Per-frame logic
    }

    /// <summary>Called every fixed timestep. Updates physics-dependent calculations.</summary>
    void FixedUpdate()
    {
        // Physics calculations
    }

    /// <summary>Called when the component is destroyed. Cleans up subscriptions and resources.</summary>
    void OnDestroy()
    {
        // Cleanup: unsubscribe events, release resources
    }

    // Public methods
    // Private methods
}
```

### Field serialization

Use `[SerializeField]` for private fields that need Inspector access. Use `[HideInInspector]`
for public fields that should not appear in the Inspector:

```csharp
/// <summary>The serialized reference to the display configuration.</summary>
[SerializeField]
private DisplayObject _display;

/// <summary>Determines whether the boundary has been disarmed. Not visible in Inspector.</summary>
[HideInInspector]
public bool boundaryDisarmed = false;
```

### Component access

Use `TryGetComponent<T>()` for safe component access. Cache component references in `Awake()`
or `Start()` instead of calling `GetComponent<T>()` in `Update()`:

```csharp
/// <summary>The cached reference to the mesh renderer component.</summary>
private MeshRenderer _meshRenderer;

/// <summary>Initializes cached component references.</summary>
void Awake()
{
    if (!TryGetComponent(out _meshRenderer))
    {
        Debug.LogWarning($"No MeshRenderer found on {gameObject.name}.");
    }
}
```

### Collider callbacks

Unity collider callbacks follow a consistent pattern:

```csharp
/// <summary>Called when the animal enters the trigger zone collider.</summary>
void OnTriggerEnter(Collider other)
{
    if (!isActive)
        return;

    _inZone = true;
    Debug.Log("Animal entered trigger zone.");
}

/// <summary>Called when the animal exits the trigger zone collider.</summary>
void OnTriggerExit(Collider other)
{
    if (!isActive)
        return;

    _inZone = false;
}
```

---

## ScriptableObject patterns

ScriptableObjects store shared data as Unity assets. Use them for configuration that multiple
components reference:

```csharp
namespace Gimbl
{
    /// <summary>Stores display configuration parameters as a reusable Unity asset.</summary>
    [System.Serializable]
    public class DisplaySettings : ScriptableObject
    {
        /// <summary>Determines whether this display is active.</summary>
        public bool isActive = true;

        /// <summary>The display brightness as a percentage (0-100).</summary>
        public float brightness = 100f;

        /// <summary>The height of the display in VR world units.</summary>
        public float heightInVR = 0.2f;
    }
}
```

### Guidelines

- Use `[System.Serializable]` attribute for Inspector serialization
- Public fields with camelCase for Inspector-editable parameters
- Inherit from `ScriptableObject` (not `MonoBehaviour`) for data-only assets
- Keep ScriptableObjects focused on data; avoid complex logic

---

## Enum conventions

### Simple enums

Use simple enums for a fixed set of named options:

```csharp
/// <summary>Defines the supported controller types for VR input devices.</summary>
public enum ControllerTypes
{
    /// <summary>A physical linear treadmill connected via serial communication.</summary>
    LinearTreadmill,

    /// <summary>A simulated treadmill for testing without physical hardware.</summary>
    SimulatedLinearTreadmill,
}
```

### Enums with explicit values

Use explicit integer values for status codes, protocol identifiers, or values that must remain
stable across versions:

```csharp
/// <summary>Defines the MQTT message type codes used by the communication protocol.</summary>
public enum MessageTypes
{
    /// <summary>A command message sent from the PC to the controller.</summary>
    Command = 1,

    /// <summary>A parameter message configuring module behavior.</summary>
    Parameters = 2,

    /// <summary>A data message sent from the controller to the PC.</summary>
    ModuleData = 6,

    /// <summary>A state message indicating module runtime status.</summary>
    ModuleState = 8,
}
```

### Rules

- **XML documentation**: Document every enum member with a `<summary>` tag
- **Class docstring**: Imperative mood ("Defines the...")
- **PascalCase**: Both enum type names and values use PascalCase
- **Trailing comma**: Include trailing comma after the last member
- **Explicit values**: Use only when stability across versions matters (protocols, serialization)

---

## Property patterns

### Expression-bodied properties

Use expression-bodied syntax for simple, single-expression properties:

```csharp
/// <summary>Determines whether this zone uses occupancy-based stimulus triggering.</summary>
public bool IsOccupancyMode => _occupancyZone != null;

/// <summary>Returns the corridor spacing in Unity units, converted from centimeters.</summary>
public float CorridorSpacingUnity => corridorSpacingCm / CmPerUnityUnit;
```

### Full property syntax

Use full property syntax when the getter or setter contains logic:

```csharp
/// <summary>Returns the display configuration object.</summary>
public DisplayObject Display
{
    get { return _display; }
    set
    {
        _display = value;
        UpdateDisplayConfiguration();
    }
}
```

### Computed properties vs methods

- Use properties for values that feel like attributes (cheap, no side effects)
- Use methods for operations that are expensive, have side effects, or take parameters

---

## Abstract class patterns

Use abstract classes for base types that define a shared interface:

```csharp
/// <summary>Base class for all VR input controllers.</summary>
public abstract class ControllerObject : MonoBehaviour
{
    /// <summary>Renders the controller's editor menu in the Inspector.</summary>
    public abstract void EditMenu();

    /// <summary>Links the controller settings from the specified asset path.</summary>
    /// <param name="assetPath">The path to the settings asset. Defaults to empty string.</param>
    public abstract void LinkSettings(string assetPath = "");
}
```

### Rules

- Mark abstract classes with `abstract` keyword
- Mark abstract methods with `abstract` keyword
- Derived classes use `override` keyword (not `new`)
- Use `sealed` on classes that should not be further derived
- Use `virtual` for methods that may optionally be overridden

---

## Generic class patterns

Use generics for type-safe reusable components:

```csharp
/// <summary>A typed MQTT channel that deserializes messages to the specified type.</summary>
/// <typeparam name="TMessage">The message type for JSON deserialization.</typeparam>
public class MQTTChannel<TMessage> : MQTTChannel
{
    /// <summary>Unity event that fires when a typed message is received.</summary>
    public class ChannelEvent : UnityEvent<TMessage> { }

    /// <summary>The event subscribers can listen to for incoming messages.</summary>
    public new ChannelEvent Event = new ChannelEvent();

    /// <summary>Creates a new typed MQTT channel with the specified topic.</summary>
    /// <param name="topicString">The MQTT topic string to subscribe to.</param>
    /// <param name="isListener">Determines whether to subscribe to incoming messages.</param>
    public MQTTChannel(
        string topicString,
        bool isListener = true,
        byte qosLevel = MqttMsgBase.QOS_LEVEL_EXACTLY_ONCE
    ) : base(topicString, isListener, qosLevel) { }

    /// <summary>Deserializes the received JSON string and invokes the typed event.</summary>
    /// <param name="messageString">The raw JSON message string.</param>
    public override void ReceivedMessage(string messageString)
    {
        TMessage message = JsonUtility.FromJson<TMessage>(messageString);
        Event.Invoke(message);
    }
}
```

### Rules

- Type parameter names use `T` prefix with PascalCase: `TMessage`, `TConfig`, `TResult`
- Document type parameters with `<typeparam>` tags
- Use constructor delegation (`: base(...)`) for derived class constructors
- Prefer generic constraints when the type parameter must implement an interface

---

## Nested class patterns

Use nested classes for types that are tightly coupled to the enclosing class:

```csharp
/// <summary>Manages the VR task execution and corridor transitions.</summary>
public class Task : MonoBehaviour
{
    /// <summary>Wraps the segment sequence data for MQTT transmission.</summary>
    public class SequenceMessage
    {
        /// <summary>The array of segment indices in the sequence.</summary>
        public int[] sequence;
    }

    /// <summary>Wraps the scene name for MQTT transmission.</summary>
    public class SceneNameMessage
    {
        /// <summary>The name of the active Unity scene.</summary>
        public string sceneName;
    }
}
```

### Guidelines

- Use nested classes for message types, event args, or helper types specific to the parent
- Keep nested classes small and focused
- Document nested classes with the same XML conventions as top-level classes
- Prefer top-level classes when the type could be reused by other classes

---

## Static utility classes

Use static classes for stateless utility functions:

```csharp
/// <summary>Provides utility methods for measuring and validating prefab geometry.</summary>
public static class Utility
{
    /// <summary>Measures the Z-axis lengths of the given segment prefabs.</summary>
    /// <param name="segmentPrefabs">The array of segment GameObjects to measure.</param>
    /// <returns>An array of measured lengths in Unity units.</returns>
    public static float[] GetSegmentLengths(GameObject[] segmentPrefabs)
    {
        float[] lengths = new float[segmentPrefabs.Length];
        for (int i = 0; i < segmentPrefabs.Length; i++)
        {
            lengths[i] = GetPrefabLength(segmentPrefabs[i]);
        }
        return lengths;
    }

    /// <summary>Measures the Z-axis length of a single prefab using its renderer bounds.</summary>
    /// <param name="prefab">The GameObject to measure.</param>
    /// <returns>The Z-axis length in Unity units.</returns>
    public static float GetPrefabLength(GameObject prefab)
    {
        Renderer[] renderers = prefab.GetComponentsInChildren<Renderer>();
        if (renderers.Length == 0)
        {
            Debug.LogWarning($"No renderers found on {prefab.name}.");
            return 0f;
        }

        Bounds bounds = renderers[0].bounds;
        for (int i = 1; i < renderers.Length; i++)
        {
            bounds.Encapsulate(renderers[i].bounds);
        }
        return bounds.size.z;
    }
}
```

### Guidelines

- Mark the class as `static` when all methods are static
- Use static utility classes for stateless operations (measuring, validation, conversion)
- Prefer instance methods on the relevant class when the operation needs state

---

## Immutability patterns

Immutability is a core design principle across all Sun Lab projects, matching the pervasive
`const` correctness in the C++ codebase.

### Readonly fields

Mark fields as `readonly` when they are only assigned in the constructor or field initializer.
This is the C# equivalent of C++ `const` member variables:

```csharp
/// <summary>Manages communication with the microcontroller.</summary>
public class Communication
{
    /// <summary>The serial port assigned at construction time.</summary>
    private readonly SerialPort _port;

    /// <summary>The baud rate for serial communication.</summary>
    private readonly int _baudRate;

    /// <summary>Initializes the communication manager with the specified port.</summary>
    /// <param name="portName">The serial port name (e.g., "COM3").</param>
    /// <param name="baudRate">The baud rate for communication.</param>
    public Communication(string portName, int baudRate)
    {
        _port = new SerialPort(portName, baudRate);
        _baudRate = baudRate;
    }
}
```

### Readonly struct

Use `readonly struct` for small, immutable value types (the C# equivalent of C++ packed
structs with all-const members):

```csharp
/// <summary>Represents a 3D position in Unity world space.</summary>
public readonly struct WorldPosition
{
    /// <summary>The X coordinate in Unity units.</summary>
    public float X { get; }

    /// <summary>The Y coordinate in Unity units.</summary>
    public float Y { get; }

    /// <summary>The Z coordinate in Unity units.</summary>
    public float Z { get; }

    /// <summary>Creates a new world position with the specified coordinates.</summary>
    public WorldPosition(float x, float y, float z)
    {
        X = x;
        Y = y;
        Z = z;
    }
}
```

### Record types

Use `record` types for immutable data with value equality semantics (the closest C# equivalent
to Python's `@dataclass(frozen=True)`):

```csharp
/// <summary>Defines a corridor segment configuration loaded from YAML.</summary>
public record SegmentConfig(string Name, float LengthCm, int CueCount);

// Usage with value equality:
var segment1 = new SegmentConfig("A", 100f, 3);
var segment2 = new SegmentConfig("A", 100f, 3);
bool equal = segment1 == segment2;  // true (value equality)

// Non-destructive mutation with 'with':
var modified = segment1 with { LengthCm = 200f };
```

### When to use each

| Pattern           | Use when                                        | C++ equivalent           |
|-------------------|-------------------------------------------------|--------------------------|
| `const`           | Compile-time constant (primitives, strings)     | `constexpr`              |
| `static readonly` | Runtime-initialized immutable value             | `static const`           |
| `readonly` field  | Instance field set only in constructor          | `const` member variable  |
| `readonly struct` | Small immutable value type with value semantics | Packed struct, all const |
| `record`          | Immutable reference type with value equality    | N/A (use struct in C++)  |
| `record struct`   | Immutable value type with value equality        | Packed const struct      |

---

## Generic constraints

Use generic constraints to enforce compile-time type safety. This is the C# equivalent of C++
`static_assert` on template parameters:

```csharp
/// <summary>Serializes data objects for MQTT transmission.</summary>
/// <typeparam name="TData">The data type, which must be a value type for binary layout.</typeparam>
public class DataSerializer<TData> where TData : struct
{
    /// <summary>Serializes the data object to a byte array.</summary>
    public byte[] Serialize(TData data)
    {
        // Implementation
    }
}

// Multiple constraints
/// <summary>Provides a typed message channel with JSON deserialization.</summary>
/// <typeparam name="TMessage">The message type with a parameterless constructor.</typeparam>
public class TypedChannel<TMessage> where TMessage : class, new()
{
}
```

### Multi-constraint formatting

When a type has multiple type parameters with constraints, place each `where` clause on its
own line:

```csharp
public class Processor<TInput, TOutput>
    where TInput : struct
    where TOutput : class, new()
{
}
```

### Available constraints

| Constraint             | Meaning                                           |
|------------------------|---------------------------------------------------|
| `where T : struct`     | T must be a value type                            |
| `where T : class`      | T must be a reference type                        |
| `where T : new()`      | T must have a parameterless constructor           |
| `where T : BaseClass`  | T must derive from BaseClass                      |
| `where T : IInterface` | T must implement IInterface                       |
| `where T : unmanaged`  | T must be an unmanaged type (no reference fields) |
| `where T : notnull`    | T must be non-nullable                            |

---

## Constructor patterns

### Constructor chaining

Use `: this(...)` for overloaded constructors and `: base(...)` for derived class constructors.
Place the colon on the same line as the closing parenthesis when it fits:

```csharp
/// <summary>Creates a channel with default QoS level.</summary>
public MQTTChannel(string topic, bool isListener = true)
    : this(topic, isListener, DefaultQosLevel)
{
}

/// <summary>Creates a channel with the specified QoS level.</summary>
public MQTTChannel(string topic, bool isListener, byte qosLevel)
    : base(topic, isListener, qosLevel)
{
    _event = new ChannelEvent();
}
```

### Constructor validation

Validate parameters at the start of constructors using guard clauses:

```csharp
/// <summary>Initializes the encoder module with the specified pins.</summary>
/// <param name="pinA">The digital pin for encoder channel A.</param>
/// <param name="pinB">The digital pin for encoder channel B.</param>
public EncoderModule(int pinA, int pinB)
{
    if (pinA == pinB)
    {
        Debug.LogError("EncoderModule PinA and PinB cannot be the same.");
        return;
    }

    _pinA = pinA;
    _pinB = pinB;
}
```

### Parameter ordering

Order constructor parameters consistently: identity parameters first, dependencies second,
optional/configuration parameters last:

```csharp
public Module(byte moduleType, byte moduleId, Communication communication, bool verbose = false)
```

---

## Virtual / override / sealed patterns

### Rules

- Use `abstract` for methods that derived classes MUST implement
- Use `virtual` for methods that derived classes MAY override
- Always use `override` (never `new`) when overriding a base method
- Use `sealed override` to prevent further overriding in derived classes
- Use `sealed` on classes that should not be further derived

```csharp
/// <summary>Base class for hardware module controllers.</summary>
public abstract class Module
{
    /// <summary>Configures the module with parameters received from the PC.</summary>
    public abstract bool SetCustomParameters();

    /// <summary>Executes the active command. Derived classes may override for custom logic.</summary>
    public virtual bool RunActiveCommand()
    {
        return false;
    }
}

/// <summary>Controls a quadrature encoder module. Cannot be further derived.</summary>
public sealed class EncoderModule : Module
{
    /// <summary>Configures the encoder with delta threshold parameters.</summary>
    public sealed override bool SetCustomParameters()
    {
        // Implementation
        return true;
    }
}
```

### Interface vs abstract class

| Use              | When                                                       |
|------------------|------------------------------------------------------------|
| `interface`      | Pure contract with no shared implementation                |
| `abstract class` | Shared state or partial implementation needed              |
| Both             | Interface for the contract, abstract class for shared code |

---

## Struct vs class guidance

| Use `struct` when                        | Use `class` when                         |
|------------------------------------------|------------------------------------------|
| Logically represents a single value      | Represents an entity with identity       |
| Instance size is under 16 bytes          | Instance size exceeds 16 bytes           |
| Immutable after creation                 | Mutable state is needed                  |
| Frequently allocated (avoid GC pressure) | Long-lived with complex lifecycle        |
| Value equality semantics needed          | Reference equality is appropriate        |

---

## ToString conventions

Implement `ToString()` for debugging output, matching the Python `__repr__` convention:

```csharp
/// <summary>Returns a string representation of the segment configuration.</summary>
public override string ToString()
{
    return $"SegmentConfig(Name={Name}, LengthCm={LengthCm}, CueCount={CueCount})";
}
```

### Rules

- Format: `ClassName(key=value, key=value)`
- Include only the most important attributes
- Use the actual class name
- Summary: "Returns a string representation of the [ClassName] instance."

---

## Method overloading

### Ordering

Overloads with fewer parameters appear before those with more parameters:

```csharp
public void QueueCommand(byte command)
public void QueueCommand(byte command, bool noblock)
public void QueueCommand(byte command, bool noblock, uint cycleDelay)
```

### Optional parameters vs overloads

Prefer optional parameters for simple default-value cases. Use overloads when the
implementations differ significantly:

```csharp
// Good - optional parameter for simple default
public void LinkSettings(string assetPath = "")

// Good - overloads when logic differs
public void Send()                    // Sends empty message
public void Send(TMessage message)    // Serializes and sends typed message
```

---

## Attribute ordering

When stacking multiple attributes on a single member, use this order (outermost to innermost):

1. **Serialization attributes**: `[System.Serializable]`, `[StructLayout]`
2. **Unity attributes**: `[SerializeField]`, `[HideInInspector]`, `[Header]`, `[Tooltip]`
3. **Editor attributes**: `[MenuItem]`, `[CustomEditor]`
4. **Validation attributes**: `[Range]`, `[Min]`, `[Max]`
5. **Custom attributes**

```csharp
[System.Serializable]
[Header("Zone Configuration")]
[Tooltip("The duration in milliseconds for occupancy detection.")]
[SerializeField]
private float _occupancyDurationMs = 1000f;
```
