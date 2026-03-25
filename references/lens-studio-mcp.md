# Lens Studio MCP — AI Agent Integration

## What It Is

Lens Studio 5.15+ ships a built-in HTTP MCP server at `localhost:50050/mcp`. The `lens-studio-mcp` Python package is a thin proxy that:

- Discovers all tools from Lens Studio's API automatically
- Registers them as MCP tools with friendly wrappers
- Forwards calls via HTTP JSON-RPC with token-based auth
- Adds convenience tools for face filters, materials, colors, and macOS UI automation

```
AI Agent ←─ MCP (stdio/SSE/HTTP) ─→ lens-studio-mcp ←─ HTTP JSON-RPC ─→ Lens Studio (:50050)
```

## Installation and Configuration

```bash
pip install lens-studio-mcp
```

Add to your MCP config (Claude Desktop, Claude Code, Cursor, etc.):

```json
{
  "mcpServers": {
    "lens-studio": {
      "command": "lens-studio-mcp",
      "args": ["--transport", "stdio"]
    }
  }
}
```

For remote/web clients use SSE transport:

```bash
lens-studio-mcp --transport sse --port 8000
```

**First connection:** Lens Studio shows a permission popup — click Allow. The auth token is cached at `~/.lens_mcp_token.json` and reused automatically.

**Prerequisite:** Lens Studio 5.15+ must be open with a project loaded before calling any tools.

---

## Standard Operating Procedure

Always follow this order when using MCP to modify a Lens Studio project:

1. **Introspect first** — call `GetLensStudioSceneGraph` and `ListLensStudioAssets` in parallel to get current UUIDs and property paths. Never guess UUIDs.
2. **Look up unknowns** — call `QueryLensStudioRag` for any API or property path you are unsure of.
3. **Execute** — make changes using the appropriate tools.
4. **Verify** — for code changes: `CompileWithLogsTool` → `RunAndCollectLogsTool` to confirm zero errors.

> **Save first.** If your changes will restructure the scene significantly, suggest the user save before you begin (`ui_save_project()`).

---

## Tool Reference

### Connection & Status

| Tool | Description |
|---|---|
| `connect()` | Authenticate with Lens Studio. Triggers permission popup on first use. Call before anything else. |
| `status()` | Returns `{authenticated, port, base_url, cached_tools}` |
| `list_tools()` | Returns all tools Lens Studio exposes (names + descriptions) |
| `lens_tool(name, arguments)` | Call any Lens Studio tool directly by name — escape hatch for tools not covered by convenience wrappers |

### Scene Introspection

| Tool | Description |
|---|---|
| `get_scene()` → `GetLensStudioSceneGraph` | Full scene hierarchy: objects, types, components, UUIDs, property paths/values. **Source of truth.** |
| `GetLensStudioSceneObjectById(objectUUID)` | Get a single object by UUID |
| `GetLensStudioSceneObjectByName(name)` | Resolve a name to its UUID and properties |
| `ListLensStudioAssets()` | All assets: IDs, types, paths, properties. **Source of truth for assets.** |
| `GetLensStudioAssetById(assetUUID)` | Get a single asset by UUID |
| `GetLensStudioAssetByPath(path)` | Get asset by file path |
| `GetLensStudioAssetsByName(name)` | Find assets by name |
| `ListInstalledPackagesTool()` | List installed .lspkg packages |
| `LensStudio()` | General environment context (version, project name, etc.) |
| `GetLensStudioContextMenuQueue()` | Items the user right-clicked to set as AI context |

### Creating Objects and Assets

Prefer **presets** over raw creation — they set up the correct component stack.

| Tool | Description |
|---|---|
| `create_object(name, preset?)` → `CreateLensStudioSceneObject` | Create a scene object, optionally from a preset |
| `add_primitive(shape, name?)` | Quick shorthand: `cube`, `sphere`, `cylinder`, `camera`, `light`, `image`, `text` |
| `GetPresetRegistryTool()` | Browse all available presets |
| `CreateSceneObjectFromPresetTool(preset, name)` | Create from a named preset (preferred over raw create) |
| `CreateComponentFromPresetTool(preset, objectUUID)` | Add a component from preset to an existing object |
| `CreateAssetFromPresetTool(preset, name)` | Create an asset from preset (e.g. `SimplePBRMaterialPreset`, `UnlitMaterialPreset`) |
| `CreateLensStudioSceneObject(name, preset?)` | Raw create (use preset tools when possible) |
| `CreateLensStudioComponent(objectUUID, componentType)` | Add a component to an object |
| `CreateLensStudioAsset(name, assetType)` → `create_asset(name, type)` | Create a new asset (e.g. `Material`, `RenderTarget`) |
| `InstantiateLensStudioPrefab(prefabUUID, parentUUID?)` | Instantiate a prefab into the scene |
| `CreatePrefabFromSceneObject(objectUUID, name)` | Save a scene object as a prefab asset |

Common preset names:
```
SphereMeshObjectPreset    BoxMeshObjectPreset    CylinderMeshObjectPreset
CameraObjectPreset        DirectionalLightObjectPreset
ScreenImageObjectPreset   ScreenTextObjectPreset
HeadBindingObjectPreset   (face tracking anchor)
SimplePBRMaterialPreset   UnlitMaterialPreset
```

### Modifying Properties

`SetLensStudioProperty` is the most important tool. It sets any property on any object, component, or asset by UUID.

```typescript
// Signature (via convenience wrapper)
set_property(
  object_uuid: string,    // UUID of object, component, or asset
  property_path: string,  // e.g. "enabled", "localTransform.position", "passInfos.0.baseColor"
  value: any,
  value_type: string      // see table below
)
```

**value_type reference:**

| valueType | Value format | Example use |
|---|---|---|
| `"number"` | `1.5` | `intensity`, `opacity`, any numeric field |
| `"string"` | `"hello"` | Text content, names |
| `"boolean"` | `true` / `false` | `enabled` on/off |
| `"enum"` | Integer index | Dropdown fields |
| `"vec2"` | `{"x":0,"y":0}` | 2D size, UV |
| `"vec3"` | `{"x":0,"y":0,"z":0}` | Position, scale, rotation |
| `"vec4"` | `{"x":0,"y":0,"z":0,"w":1}` | Color (RGBA 0-1), quaternion |
| `"transform"` | `{"position":{x,y,z},"rotation":{x,y,z},"scale":{x,y,z}}` | Full local transform |
| `"reference"` | UUID string or `"null"` | Material, texture, asset links |
| `"layer_set_mask"` | Numeric bitmask | Layer visibility |

**Common property paths:**

```
enabled                              → object on/off
localTransform                       → full transform (use "transform" valueType)
localTransform.position              → position vec3
worldTransform.position              → world-space position vec3
mainMaterial                         → material reference on RenderMeshVisual
passInfos.0.baseColor                → RGBA color on Material (vec4, 0-1 range)
passInfos.0.baseTex                  → texture reference on Material
passInfos.0.metallic                 → metallic factor (number 0-1)
passInfos.0.roughness                → roughness factor (number 0-1)
```

**Script `@input` fields** — use the `@input` variable name directly as `propertyPath`:

```typescript
// TypeScript @input
@input foxPrefab: ObjectPrefab
@input targetObjects: SceneObject[]  // array: use "targetObjects.0", "targetObjects.1"
```

```python
# Assign via MCP
set_property(script_component_uuid, "foxPrefab", fox_asset_uuid, "reference")
set_property(script_component_uuid, "targetObjects.0", obj1_uuid, "reference")
```

### Scene Operations

| Tool | Description |
|---|---|
| `SetLensStudioParent(objectUUID, parentUUID?)` | Reparent an object. Omit `parentUUID` to move to scene root. |
| `RenameLensStudioSceneObject(objectUUID, name)` | Rename an object |
| `DuplicateLensStudioSceneObject(objectUUID)` | Duplicate an object |
| `DeleteLensStudioSceneObject(objectUUID)` → `delete_object(name)` | Remove an object |

> **Critical:** Reparenting to scene root (omit `parentUUID`) fixes the camera-following bug. See `world-space-anchoring.md`.

### Asset Operations

| Tool | Description |
|---|---|
| `RenameAsset(assetUUID, name)` | Rename an asset |
| `MoveLensStudioAsset(assetUUID, path)` | Move asset to a different folder |
| `DuplicateLensStudioAsset(assetUUID)` | Duplicate an asset |
| `DeleteLensStudioAsset(assetUUID)` | Delete an asset |
| `ReadWriteTextFile(path, content?)` | Read or write a file on disk — **primary tool for TypeScript authoring** |

### Scripting Workflow

The TypeScript-first workflow for any code task:

```
1. ReadWriteTextFile("Assets/Scripts/MyScript.ts", content)  → create/update .ts file
2. CompileWithLogsTool()                                       → must succeed before attaching
3. CreateLensStudioComponent(objectUUID, "ScriptComponent")   → add ScriptComponent
4. SetLensStudioProperty(compUUID, "scriptAsset", tsAssetUUID, "reference")  → link TS file
5. RunAndCollectLogsTool()                                     → verify zero runtime errors
```

**Critical rules:**
- Scripts are **assets** — create via `ReadWriteTextFile`, then attach via `ScriptComponent`
- Always `CompileWithLogsTool` after editing code — never skip
- Always `RunAndCollectLogsTool` to catch runtime errors
- Use `print()` for logging inside scripts, not `console.log`
- Cross-object access in `onAwake` is unsafe — use `OnStartEvent` instead

| Tool | Description |
|---|---|
| `CompileWithLogsTool()` | Compile TypeScript. Fix all errors before proceeding. |
| `RunAndCollectLogsTool()` | Restart preview and collect runtime logs |
| `GetLensStudioLogsTool()` | Read logs without restarting preview |

### Asset Library

| Tool | Description |
|---|---|
| `SearchLensStudioAssetLibrary(query)` | Search the Asset Library |
| `InstallLensStudioPackage(packageId)` | Install a package from the Asset Library |
| `SearchLensStudioMusicLibrary(query)` | Search licensed music |
| `InstallLicensedMusic(trackId)` | Install a music track |

### AI Generators

| Tool | Description |
|---|---|
| `GenerateFast3DAssets(prompt)` | Generate 3D assets from text description |
| `GenerateThreeDAssetTool(prompt)` | Higher-quality 3D asset generation |
| `GenerateTexture(prompt, size?)` | Generate a texture image |
| `GenerateFaceMaskTexture(prompt)` | Generate a face-mask-optimised texture |

### Knowledge Base

| Tool | Description |
|---|---|
| `QueryLensStudioRag(query)` → `query_knowledge_base(query)` | Search Lens Studio docs. Use before writing code or setting properties you're unsure about. |

### Materials and Colors (Convenience Wrappers)

| Tool | Description |
|---|---|
| `create_material(name, color?, r?, g?, b?, a?, metallic?, roughness?, unlit?)` | Create PBR or unlit material with a solid color |
| `set_color(name, color?, r?, g?, b?)` | Set a mesh object's color — creates material automatically |
| `assign_material(object_name, material_name)` | Assign an existing material to an object |

Color can be specified as:
- Named color: `"red"`, `"blue"`, `"pink"`, `"gold"`, `"cyan"`, etc.
- Hex: `"#FF6600"` or `"#F60"`
- RGB floats: `r=1.0, g=0.5, b=0.0`

### Face Filters (Convenience Wrappers)

| Tool | Description |
|---|---|
| `create_face_anchor(name?)` | Create HeadBindingObjectPreset with zeroed localTransform + Face Occluder |
| `add_face_element(parent_name, landmark, shape?, name?, scale?, rotation?)` | Add a 3D element at a face landmark, parented to the anchor |
| `get_face_landmarks(landmark?)` | Reference positions for `nose_tip`, `left_ear_top`, `forehead`, `chin`, etc. |

> **Critical for face filters:** The Head Anchor's `localTransform` **must** be `{0,0,0}`. Any non-zero offset shifts ALL children. The `create_face_anchor()` wrapper zeroes it automatically.

Face landmarks (local position offsets from HeadCenter):

| Landmark | Position (x, y, z) | Notes |
|---|---|---|
| `nose_tip` | (0, 5, 9.5) | Center of face, front |
| `nose_bridge` | (0, 7.5, 8) | Between eyes |
| `forehead` | (0, 12, 6) | Center forehead |
| `chin` | (0, -3, 6) | Bottom of chin |
| `mouth_center` | (0, 2, 8.5) | Center mouth |
| `left_eye` | (-3, 7.5, 7.5) | |
| `right_eye` | (3, 7.5, 7.5) | |
| `left_ear_top` | (-7.4, 16.4, -2) | Cat/animal ear |
| `right_ear_top` | (7.4, 16.4, -2) | Cat/animal ear |
| `left_whiskers` | (-3, 5.7, 13.5) | |
| `crown` | (0, 18, 0) | Hats, horns |

### macOS UI Automation (Fallback)

For operations the API doesn't cover. macOS only.

| Tool | Description |
|---|---|
| `ui_activate()` | Bring Lens Studio to foreground |
| `ui_menu_click(menu, item)` | Click a menu item (e.g. `"File"`, `"Save"`) |
| `ui_new_project()` | File > New Project |
| `ui_save_project()` | File > Save |
| `ui_open_project(path)` | Open a .lsproj file |
| `ui_preview(action)` | `"start"`, `"stop"`, or `"reload"` preview |
| `ui_keystroke(key, modifiers?)` | Send keystroke (e.g. `key="s"`, `modifiers=["cmd"]`) |
| `ui_type_text(text)` | Type text into the frontmost field |
| `ui_import_asset(path)` | File > Import + navigate dialog |
| `ui_export_lens(path)` | File > Export + navigate dialog |
| `ui_add_object(kind)` | Object menu: `"sprite"`, `"text"`, `"empty"` |
| `ui_dump_menus()` | List all menu items — use to discover exact item names |
| `ui_list_buttons(substring?)` | List buttons in the frontmost window |
| `ui_click_button(name)` | Click a named button |
| `ui_request_permissions()` | Trigger macOS Automation permissions popup (run once on setup) |

---

## Common Workflows

### Reparent an object to scene root (fix camera-following bug)

```python
# 1. Get the object UUID
scene = get_scene()   # read the response to find UUID

# 2. Reparent to scene root (omit parentUUID)
lens_tool("SetLensStudioParent", {"objectUUID": "a1b2c3..."})
```

### Change a text label

```python
# 1. Find the Text component UUID via get_scene()
# 2. Set text content
lens_tool("SetLensStudioProperty", {
    "objectUUID": "text-component-uuid",
    "propertyPath": "text",
    "value": "Hello World",
    "valueType": "string",
})
```

### Enable/disable a scene object

```python
lens_tool("SetLensStudioProperty", {
    "objectUUID": "object-uuid",
    "propertyPath": "enabled",
    "value": False,
    "valueType": "boolean",
})
```

### Set world position

```python
lens_tool("SetLensStudioProperty", {
    "objectUUID": "object-uuid",
    "propertyPath": "localTransform",
    "value": {
        "position": {"x": 0, "y": -30, "z": -50},
        "rotation": {"x": 0, "y": 0, "z": 0},
        "scale": {"x": 20, "y": 20, "z": 20},
    },
    "valueType": "transform",
})
```

### Create and attach a TypeScript script

```python
# 1. Write the TypeScript file
ReadWriteTextFile("Assets/Scripts/MyBehaviour.ts", ts_source_code)

# 2. Compile
CompileWithLogsTool()

# 3. Add ScriptComponent to the target object
comp_result = CreateLensStudioComponent(object_uuid, "ScriptComponent")

# 4. Get the TS asset UUID
ts_asset = GetLensStudioAssetByPath("Assets/Scripts/MyBehaviour.ts")

# 5. Link TS file to the ScriptComponent
SetLensStudioProperty(comp_uuid, "scriptAsset", ts_asset_uuid, "reference")

# 6. Run and verify
RunAndCollectLogsTool()
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Unauthorized access attempt` | No auth token, or token expired | Call `connect()` — click Allow in the Lens Studio popup |
| `Cannot connect to Lens Studio` | LS not running or wrong port | Start Lens Studio 5.15+; check `LENS_MCP_PORT` env var |
| `Object not found` | Wrong name or not yet in scene | Use `get_scene()` to verify exact name/UUID |
| Property set has no visible effect | Wrong `valueType` | Check `QueryLensStudioRag` for the exact type |
| Script compiles but doesn't run | ScriptComponent not linked | Verify step 4 (link `scriptAsset`) in scripting workflow |
| Face filter elements misaligned | Head Anchor has non-zero transform | Zero the Head Anchor's `localTransform` |
| UI automation fails with permission error | macOS Accessibility not granted | Run `ui_request_permissions()` and click Allow |
