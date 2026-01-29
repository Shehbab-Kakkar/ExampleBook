Perfect — you already did the most important step by running `helm template`.
Now let’s **trace exactly how data flows** from `values.yaml` → template → **two ConfigMaps**, line by line, with *no magic skipped*.

---

# 1️⃣ What Helm has before rendering

When you run:

```bash
helm template configmap-example ./configmap-example --dry-run
```

Helm builds a **root rendering context** called `.`

Conceptually:

```
.
├── Chart
│   ├── Name = "configmap-example"
│   └── Version = "0.1.0"
├── Release
│   ├── Name = "configmap-example"
│   └── Namespace = "default"
├── Values
│   └── configMaps
│       ├── app-config
│       │   ├── LOG_LEVEL = "info"
│       │   └── TIMEOUT   = "30"
│       └── db-config
│           ├── DB_HOST = "postgres"
│           └── DB_PORT = "5432"
└── Capabilities
```

This `.` is what every template starts with.

---

# 2️⃣ `$root := .` (locking the root context)

```gotemplate
{{- $root := . -}}
```

This line **copies the entire root context** into `$root`.

Why this matters:

* `range` will **change what `.` means**
* `$root` never changes

After this line:

```
$root → full Helm context (Chart, Release, Values)
.     → still the full context (for now)
```

---

# 3️⃣ Outer `range`: looping over `configMaps`

```gotemplate
{{- range $name, $data := .Values.configMaps }}
```

Helm now evaluates:

```yaml
.Values.configMaps
```

Which is this map:

```yaml
app-config:
  LOG_LEVEL: info
  TIMEOUT: "30"
db-config:
  DB_HOST: postgres
  DB_PORT: "5432"
```

### What `range` does internally

For **each entry** in the map:

| Variable | Value                               |
| -------- | ----------------------------------- |
| `$name`  | map key (`app-config`, `db-config`) |
| `$data`  | map value (inner key/value pairs)   |
| `.`      | **same as `$data`**                 |

⚠️ This is critical:

> Inside `range`, `.` is no longer the root — it becomes the **current map item**

---

## First iteration (`app-config`)

```
$name = "app-config"
$data = {
  LOG_LEVEL: "info",
  TIMEOUT: "30"
}
. = $data
```

---

# 4️⃣ Creating the ConfigMap object

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ $name }}
```

Rendered as:

```yaml
metadata:
  name: app-config
```

This is how **the ConfigMap name is derived from values.yaml**.

---

# 5️⃣ Labels: why `$root` is required

```yaml
labels:
  app.kubernetes.io/name: {{ $root.Chart.Name }}
  app.kubernetes.io/instance: {{ $root.Release.Name }}
```

Why not `.Chart.Name`?

Because:

* `.` = `$data` (LOG_LEVEL, TIMEOUT)
* `$data.Chart` ❌ does not exist

So Helm correctly uses:

```
$root.Chart.Name    → "configmap-example"
$root.Release.Name → "configmap-example"
```

Which renders:

```yaml
labels:
  app.kubernetes.io/name: configmap-example
  app.kubernetes.io/instance: configmap-example
```

---

# 6️⃣ Inner `range`: populating ConfigMap `data`

```gotemplate
{{- range $key, $value := $data }}
  {{ $key }}: {{ $value | quote }}
{{- end }}
```

Now Helm loops over the **inner map**:

```yaml
LOG_LEVEL: info
TIMEOUT: "30"
```

Iteration by iteration:

| `$key`    | `$value` |
| --------- | -------- |
| LOG_LEVEL | info     |
| TIMEOUT   | "30"     |

Rendered YAML:

```yaml
data:
  LOG_LEVEL: "info"
  TIMEOUT: "30"
```

Why `| quote`?

* Forces strings (ConfigMap data **must be strings**)
* Avoids YAML type confusion

---

# 7️⃣ YAML document separator `---`

```gotemplate
---
```

This tells Kubernetes:

> “Next resource starts here”

Without this, you’d generate **invalid YAML** when multiple ConfigMaps exist.

---

# 8️⃣ Second iteration (`db-config`)

Now Helm repeats **everything** inside the outer `range`.

```
$name = "db-config"
$data = {
  DB_HOST: "postgres",
  DB_PORT: "5432"
}
```

This produces:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  DB_HOST: "postgres"
  DB_PORT: "5432"
```

---

# 9️⃣ Why Helm prints `# Source: ...`

```yaml
# Source: configmap-example/templates/configmap.yaml
```

Helm adds this comment so you know:

* Which template file generated which resource
* Extremely useful for debugging large charts

---

# 🔁 End-to-end data flow (visual)

```
values.yaml
   ↓
.Values.configMaps
   ↓ (outer range)
$name  → ConfigMap name
$data  → map of data entries
   ↓ (inner range)
$key   → data key
$value → data value
   ↓
Rendered ConfigMap YAML
```

---

# 🧠 Key takeaways (this is the Helm “aha” moment)

✅ `values.yaml` **drives resource creation**
✅ `range` over a map → **multiple resources**
✅ `.` changes inside `range`
✅ `$root` preserves access to Chart & Release
✅ Inner `range` fills `data:` cleanly
✅ `---` separates Kubernetes objects

---

If you want next, I can show:

* ❌ What breaks if you remove `$root`
* 🧪 How `helm install` differs from `helm template`
* 🔍 How overrides (`-f`, `--set`) change this flow
* 🧠 Rewrite using `_helpers.tpl`

Just say 👍
