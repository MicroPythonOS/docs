# NumberFormat

The NumberFormat framework provides a small utility for formatting numbers using the system's number format preference stored in SharedPreferences.

## Preferences

The format preference is stored in `SharedPreferences` under:

- App ID: `com.micropythonos.settings`
- Key: `number_format`

If the key is missing, the default format is `comma_dot`.

## Supported formats

The available formats are:

- `comma_dot`: `1,234.56` (US/UK)
- `dot_comma`: `1.234,56` (Europe)
- `space_comma`: `1 234,56` (French)
- `apos_dot`: `1'234.56` (Swiss)
- `under_dot`: `1_234.56` (Tech)
- `none_dot`: `1234.56` (no thousands separator)
- `none_comma`: `1234,56` (no thousands separator)

## API

### `NumberFormat.refresh_preference()`

Reloads the preference from SharedPreferences.

### `NumberFormat.get_separators()`

Returns a tuple `(decimal_sep, thousands_sep)` for the current preference.

### `NumberFormat.format_number(value, decimals=None)`

Formats a number using the current preference.

- `value`: `int` or `float`
- `decimals`: number of decimal places
  - `None` (default):
    - `int` values are formatted without decimals
    - `float` values are formatted to 2 decimals, then trailing zeros are stripped

### `NumberFormat.get_format_options()`

Returns a list of `(label, key)` tuples for use in settings dropdowns.

## Example

```python
from mpos import NumberFormat

NumberFormat.refresh_preference()

print(NumberFormat.format_number(1234567))      # 1,234,567 (depends on preference)
print(NumberFormat.format_number(1234.5))       # 1,234.5  (depends on preference)
print(NumberFormat.format_number(1234.5, 3))    # 1,234.500 (depends on preference)
```
