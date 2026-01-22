---
applyTo: "**/*.ps1, **/*.psm1, **/*.psd1"
---
# PowerShell Coding Standards

## Core Principle: Pipeline-First

PowerShell is built around the pipeline. Use it instead of loops.

```powershell
# BAD - imperative
$results = @()
foreach ($item in $data) {
    if ($item.Status -eq 'Active') {
        $results += $item
    }
}

# GOOD - pipeline
$results = $data | Where-Object Status -eq 'Active'
```

## Function Design

### Small, Single-Purpose Functions

Each function does ONE thing. If writing multiple loops doing different things, split into separate functions.

```powershell
# GOOD - composable functions
Get-Data |
    Select-ActiveItems |
    ConvertTo-NormalizedFormat |
    Export-ToTarget
```

### Pipeline-Compatible (begin/process/end)

```powershell
function ConvertTo-NormalizedItem {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline)]
        [ValidateNotNull()]
        [object]$InputObject
    )

    process {
        [PSCustomObject]@{
            Name = $InputObject.Name.Trim().ToUpper()
            Date = [datetime]::Parse($InputObject.Date)
        }
    }
}
```

### Always Use [CmdletBinding()] and Proper Parameters

```powershell
function Get-ProcessedData {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, Position = 0)]
        [ValidateNotNullOrEmpty()]
        [string]$Path,

        [Parameter()]
        [ValidateSet('JSON', 'CSV', 'XML')]
        [string]$Format = 'JSON',

        [Parameter()]
        [switch]$IncludeMetadata
    )
    # ...
}
```

## Collections - NEVER Use $array += $item

This is O(n²) because it creates a new array every iteration.

```powershell
# BAD - O(n²)
$results = @()
foreach ($item in $data) {
    $results += $item
}

# GOOD - Generic List
$results = [System.Collections.Generic.List[object]]::new()
foreach ($item in $data) {
    $results.Add($item)
}

# BETTER - Pipeline (no explicit collection)
$results = $data | ForEach-Object { <# transform #> $_ }
```

## Error Handling

```powershell
function Get-ConfigurationFile {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string]$Path)

    try {
        if (-not (Test-Path -Path $Path -PathType Leaf)) {
            throw [System.IO.FileNotFoundException]::new("Config not found: $Path")
        }
        Get-Content -Path $Path -Raw -ErrorAction Stop | ConvertFrom-Json -ErrorAction Stop
    }
    catch [System.IO.FileNotFoundException] {
        Write-Error "Configuration file not found: $Path"
        throw
    }
    catch {
        Write-Error "Failed to read configuration: $_"
        throw
    }
}
```

## Output

Return **objects**, not formatted strings:

```powershell
# BAD
function Get-DiskInfo { "Disk C: has 50GB free" }

# GOOD
function Get-DiskInfo {
    [PSCustomObject]@{
        Drive = 'C:'
        FreeGB = 50
        TotalGB = 256
    }
}
```

Use Write-Verbose for diagnostics, not Write-Host:

```powershell
function Invoke-Processing {
    [CmdletBinding()]
    param($Data)

    Write-Verbose "Processing $($Data.Count) items"
    # ...
}
```

## Naming

- **Approved verbs**: Get-, Set-, New-, Remove-, Invoke-, Test-, ConvertTo-, ConvertFrom-
- **Functions**: PascalCase (Verb-Noun)
- **Variables**: $camelCase
- **No aliases in scripts**: Use `Where-Object` not `?`, `ForEach-Object` not `%`

## Anti-Patterns to Avoid

| Anti-Pattern | Use Instead |
|--------------|-------------|
| `$arr += $item` | `[List[T]]` or pipeline |
| `Write-Host` for data | Return objects |
| Aliases (`?`, `%`, `select`) | Full cmdlet names |
| One giant function | Small composable functions |
| No error handling | try/catch with -ErrorAction Stop |
| Hardcoded paths | Parameters with validation |
| Nested loops for filter/transform | Pipeline with Where-Object, ForEach-Object |
