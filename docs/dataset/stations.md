# Station Configuration

Stations define the individual frequencies, services, or logical sectors within an FIR. They represent _what can be called_ over the voice system (e.g. Delivery, Ground, Tower, Approach sectors), independently of how controllers log in on VATSIM.

Station configuration is stored in stations.toml or stations.json inside an FIR's dataset directory.

## File Location

```
dataset/{FIR_CODE}/stations.{toml|json}
```

## File Structure

The file must contain a single top-level array named `stations`.

TOML

```toml
[[stations]]
# Station 1 definition

[[stations]]
# Station 2 definition
```

JSON representation:

```json
{
  "stations": [
    {
      /* Station 1 definition */
    },
    {
      /* Station 2 definition */
    }
  ]
}
```

## Station Fields

Each station entry describes one callable station.

| Field           | Type             | Required      | Description                                                                                                          |
| :-------------- | :--------------- | :------------ | :------------------------------------------------------------------------------------------------------------------- |
| `id`            | String           | Yes           | Globally unique station identifier (e.g., `LOWW_TWR`, `LOVV_N1`). Must start with the FIR's two-letter country code. |
| `parent_id`     | String           | No            | Optional parent station ID, used to express hierarchical relationships between stations.                             |
| `controlled_by` | Array of strings | Conditionally | List of position IDs that explicitly control this station. Required unless `parent_id` is set.                       |

## Validation Rules

The following validation rules apply:

- `id` must be unique across the entire dataset
- `controlled_by`:
  - must contain at least one position if present
  - must not contain duplicates
- If a station does not define `parent_id`, it must define `controlled_by`
- If a station defines `parent_id`, `controlled_by` becomes optional

## Coverage Priority (`controlled_by`)

The `controlled_by` field defines the **coverage order** for a station.

It is an ordered list of Position IDs, evaluated from start to end:

- The first online position in the list is considered to be covering the station
- If the highest-priority position goes offline, coverage falls back to the next one
- If multiple listed positions are online at the same time, only the first matching one is used

This field is used for **coverage calculations**.

## Station Inheritance (`parent_id`)

The `parent_id` field allows a station to **inherit coverage priority** from another station.

Inheritance works by **appending** the resolved `controlled_by` list of the parent station to the station's own `controlled_by` list.

This process is transitive: if the parent station itself has a `parent_id`, its parent's coverage list is appended as well.

## Coverage Resolution

The effective coverage list for a station is resolved as follows:

1. Start with the station's own `controlled_by` list (if defined)
2. If `parent_id` is set:
   - Resolve the parent station's coverage list
   - Append it to the station's list
3. Repeat recursively until no `parent_id` remains
4. The final list is evaluated in order for online positions

## Examples

### Station controlled by a single position

TOML

```toml
[[stations]]
id = "LOVV_I_CTR"
controlled_by = ["LOVV_I_CTR"]
```

JSON

```json
{
  "stations": [
    {
      "id": "LOVV_I_CTR",
      "controlled_by": ["LOVV_I_CTR"]
    }
  ]
}
```

### Stations controlled by multiple positions

TOML

```toml
[[stations]]
id = "LOWW_F_APP"
controlled_by = ["LOWW_F_APP", "LOWW_D_APP"]

[[stations]]
id = "LOWW_D_APP"
controlled_by = ["LOWW_D_APP", "LOWW_F_APP"]
```

JSON

```json
{
  "stations": [
    {
      "id": "LOWW_F_APP",
      "controlled_by": ["LOWW_F_APP", "LOWW_D_APP"]
    },
    {
      "id": "LOWW_D_APP",
      "controlled_by": ["LOWW_D_APP", "LOWW_F_APP"]
    }
  ]
}
```

### Inheritance without explicit additional coverage

TOML

```toml
[[stations]]
id = "LOVV_N2"
parent_id = "LOVV_N3"

[[stations]]
id = "LOVV_N3"
parent_id = "LOVV_N4"

[[stations]]
id = "LOVV_N4"
parent_id = "LOVV_N5"

[[stations]]
id = "LOVV_N5"
parent_id = "LOVV_N6"

[[stations]]
id = "LOVV_N6"
parent_id = "LOVV_N7"

[[stations]]
id = "LOVV_N7"
controlled_by = [
    "LOVV_NU_CTR",
    "LOVV_EU_CTR",
    "LOVV_U_CTR",
    "LOVV_N_CTR",
    "LOVV_E_CTR",
    "LOVV_CTR",
    "LOVV_C_CTR",
]
```

JSON

```json
{
  "stations": [
    {
      "id": "LOVV_N2",
      "parent_id": "LOVV_N3"
    },
    {
      "id": "LOVV_N3",
      "parent_id": "LOVV_N4"
    },
    {
      "id": "LOVV_N4",
      "parent_id": "LOVV_N5"
    },
    {
      "id": "LOVV_N5",
      "parent_id": "LOVV_N6"
    },
    {
      "id": "LOVV_N6",
      "parent_id": "LOVV_N7"
    },
    {
      "id": "LOVV_N7",
      "controlled_by": [
        "LOVV_NU_CTR",
        "LOVV_EU_CTR",
        "LOVV_U_CTR",
        "LOVV_N_CTR",
        "LOVV_E_CTR",
        "LOVV_CTR",
        "LOVV_C_CTR"
      ]
    }
  ]
}
```

### Inheritance with additional coverage

TOML

```toml
[[stations]]
id = "LOWW_DEL"
parent_id = "LOWW_GND"
controlled_by = ["LOWW_DEL"]

[[stations]]
id = "LOWW_GND"
parent_id = "LOWW_TWR"
controlled_by = ["LOWW_GND", "LOWW_W_GND", "LOWW_E_TWR", "LOWW_TWR"]

[[stations]]
id = "LOWW_TWR"
parent_id = "LOWW_APP"
controlled_by = ["LOWW_TWR", "LOWW_E_TWR"]
```

JSON

```json
{
  "stations": [
    {
      "id": "LOWW_DEL",
      "parent_id": "LOWW_GND",
      "controlled_by": ["LOWW_DEL"]
    },
    {
      "id": "LOWW_GND",
      "parent_id": "LOWW_TWR",
      "controlled_by": ["LOWW_GND", "LOWW_W_GND", "LOWW_E_TWR", "LOWW_TWR"]
    },
    {
      "id": "LOWW_TWR",
      "parent_id": "LOWW_APP",
      "controlled_by": ["LOWW_TWR", "LOWW_E_TWR"]
    }
  ]
}
```
