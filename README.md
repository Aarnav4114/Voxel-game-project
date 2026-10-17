import os, textwrap, zipfile

root = "/mnt/data/FourRealmsUnityStarter"
files = {
"README.md": r"""
# Four Realms — Unity Voxel Survival Starter

A small, original Unity 3D starter project for a Minecraft-inspired voxel survival sandbox. It includes a procedural block terrain, first-person movement, block breaking/placing, survival and creative modes, a hotbar, basic crafting, hostile slimes, a villager-like NPC, melee combat, a boss, and four realm portals.

This is a **playable prototype scaffold**, not a finished commercial game. The included editor bootstrapper creates a demo scene from primitive meshes, so no paid assets are required.

## Requirements
- Unity 2022.3 LTS or newer (3D Core project)
- Desktop keyboard and mouse

## Quick start
1. In Unity Hub, create a new **3D Core** project named `FourRealms`.
2. Close Unity.
3. Copy the `Assets` folder from this package into the new project's folder, merging if asked.
4. Reopen the project and wait for scripts to compile.
5. In Unity, select **Four Realms > Create Playable Demo Scene** from the top menu.
6. Open `Assets/Scenes/FourRealmsDemo.unity` if it did not open automatically.
7. Press **Play**.

## Controls
- WASD: move
- Mouse: look (click the Game view first)
- Space: jump
- Left Shift: sprint
- Left mouse: mine block / attack
- Right mouse: place selected block
- 1–5: select hotbar slot
- E: toggle Survival / Creative
- C: craft a basic block bundle (survival mode)
- F: interact with a nearby realm portal
- Esc: release mouse

## Included systems
- Four stylized realms: Overworld, Nether, Aether and End, with portals to switch between them.
- Procedurally generated, simple block terrain and realm-specific palettes.
- First-person controller with sprint and jump.
- Block mining and placement with a small inventory/hotbar.
- Survival health, basic crafting, and creative mode with unlimited blocks.
- Slime enemies that chase and damage the player.
- Villager NPCs as village markers/ambient characters.
- Melee attacks, health bars represented through simple UI text, and a boss enemy.
- Editor menu to create a demo scene with lighting, camera, player, terrain, mobs and portals.

## Notes / limitations
- The terrain is generated from Unity cube primitives for clarity and ease of editing; it is not chunk-meshed or optimized for large worlds.
- Textures are represented by procedural materials/colors. Replace them with your own pixel textures/materials for richer visuals.
- This starter uses Unity's built-in Input APIs (`Input.GetAxis`, `Input.GetKey`). If the project is configured for the new Input System only, set **Project Settings > Player > Active Input Handling** to **Both** or **Input Manager (Old)**.
- The bootstrapper generates a demo scene and primitive geometry, not imported assets or a polished release build.
- Use original names/artwork and your own assets when expanding the game.
""",
"Assets/Scripts/RealmType.cs": r"""
public enum RealmType
{
    Overworld,
    Nether,
    Aether,
    End
}
""",
"Assets/Scripts/RealmManager.cs": r"""
using UnityEngine;
using UnityEngine.SceneManagement;

public class RealmManager : MonoBehaviour
{
    public static RealmManager Instance { get; private set; }
    public RealmType CurrentRealm { get; private set; } = RealmType.Overworld;
    public Transform player;
    public VoxelWorld world;
    public string statusMessage = "Overworld — gather, build, survive";
    private float messageUntil;

    void Awake()
    {
        if (Instance != null && Instance != this) { Destroy(gameObject); return; }
        Instance = this;
    }

    public void SwitchRealm(RealmType realm)
    {
        CurrentRealm = realm;
        if (world != null) world.Generate(realm);
        if (player != null) player.position = new Vector3(0, 12, 0);
        statusMessage = realm + " — realm changed";
        messageUntil = Time.time + 4f;
        if (HUDController.Instance != null) HUDController.Instance.ShowMessage(statusMessage);
    }

    public void SetMessage(string message)
    {
        statusMessage = message;
        messageUntil = Time.time + 3f;
        if (HUDController.Instance != null) HUDController.Instance.ShowMessage(message);
    }
}
""",
"Assets/Scripts/VoxelWorld.cs": r"""
using UnityEngine;
using System.Collections.Generic;

public class VoxelWorld : MonoBehaviour
{
    public int width = 36;
    public int depth = 36;
    public int baseHeight = 3;
    public Material grass, dirt, stone, sand, nether, aether, endStone, wood, leaves;
    private readonly Dictionary<Vector3Int, GameObject> blocks = new Dictionary<Vector3Int, GameObject>();
    private RealmType realm = RealmType.Overworld;

    public void Generate(RealmType type)
    {
        realm = type;
        foreach (var block in blocks.Values) if (block) Destroy(block);
        blocks.Clear();
        var old = transform.Find("Generated");
        if (old) Destroy(old.gameObject);
        var holder = new GameObject("Generated").transform;
        holder.SetParent(transform, false);
        for (int x = -width / 2; x < width / 2; x++)
        for (int z = -depth / 2; z < depth / 2; z++)
        {
            float n = Mathf.PerlinNoise((x + 100f) * .12f, (z + 100f) * .12f);
            int h = baseHeight + Mathf.RoundToInt(n * 3f);
            for (int y = 0; y <= h; y++)
            {
                Material m = y == h ? TopMaterial(type) : (y >= h - 2 ? MidMaterial(type) : DeepMaterial(type));
                CreateBlock(new Vector3Int(x, y, z), m, holder);
            }
        }
        // Small trees in the Overworld and Aether.
        if (type == RealmType.Overworld || type == RealmType.Aether)
        {
            for (int i = 0; i < 13; i++)
            {
                int x = Random.Range(-width/2 + 3, width/2 - 3);
                int z = Random.Range(-depth/2 + 3, depth/2 - 3);
                int h = baseHeight + 2;
                for (int y = h; y < h + 3; y++) CreateBlock(new Vector3Int(x, y, z), wood, holder);
                for (int dx=-2; dx<=2; dx++) for (int dz=-2; dz<=2; dz++)
                for (int dy=2; dy<=4; dy++)
                    if (Mathf.Abs(dx)+Mathf.Abs(dz)+(dy==4?1:0) < 5)
                        CreateBlock(new Vector3Int(x+dx, h+dy, z+dz), leaves, holder);
            }
        }
        // Village huts on the Overworld.
        if (type == RealmType.Overworld)
        {
            BuildHut(new Vector3Int(-8, baseHeight + 1, 7), holder);
            BuildHut(new Vector3Int(8, baseHeight + 1, -8), holder);
        }
        if (RealmManager.Instance != null) RealmManager.Instance.SetMessage(type + " generated");
    }

    void BuildHut(Vector3Int p, Transform holder)
    {
        for (int x=0; x<5; x++) for (int z=0; z<5; z++)
        {
            CreateBlock(p + new Vector3Int(x,0,z), wood, holder);
            for (int y=1; y<4; y++)
            {
                if (x==0 || x==4 || z==0 || z==4)
                    if (!(x==2 && z==0 && y<3)) CreateBlock(p + new Vector3Int(x,y,z), dirt, holder);
            }
        }
        for (int x=-1; x<=5; x++) for (int z=-1; z<=5; z++)
            CreateBlock(p + new Vector3Int(x,4,z), wood, holder);
    }

    Material TopMaterial(RealmType t)
    {
        switch(t) { case RealmType.Nether: return nether; case RealmType.Aether: return aether; case RealmType.End: return endStone; default: return grass; }
    }
    Material MidMaterial(RealmType t)
    {
        switch(t) { case RealmType.Nether: return nether; case RealmType.Aether: return aether; case RealmType.End: return endStone; default: return dirt; }
    }
    Material DeepMaterial(RealmType t)
    {
        switch(t) { case RealmType.Nether: return stone; case RealmType.Aether: return aether; case RealmType.End: return stone; default: return stone; }
    }

    public GameObject CreateBlock(Vector3Int pos, Material material, Transform parent = null)
    {
        var go = GameObject.CreatePrimitive(PrimitiveType.Cube);
        go.name = "Block_" + pos.x + "_" + pos.y + "_" + pos.z;
        go.transform.position = pos;
        go.transform.SetParent(parent != null ? parent : transform, true);
        go.GetComponent<Renderer>().sharedMaterial = material;
        var marker = go.AddComponent<BlockMarker>();
        marker.world = this;
        marker.gridPosition = pos;
        blocks[pos] = go;
        return go;
    }

    public void RemoveBlock(BlockMarker marker)
    {
        if (marker == null) return;
        blocks.Remove(marker.gridPosition);
        Destroy(marker.gameObject);
    }

    public Material GetBuildMaterial(int index)
    {
        switch(index % 5) { case 0: return grass; case 1: return dirt; case 2: return stone; case 3: return wood; default: return leaves; }
    }
}
""",
"Assets/Scripts/BlockMarker.cs": r"""
using UnityEngine;
public class BlockMarker : MonoBehaviour
{
    public VoxelWorld world;
    public Vector3Int gridPosition;
}
""",
"Assets/Scripts/PlayerController.cs": r"""
using UnityEngine;

[RequireComponent(typeof(CharacterController))]
public class PlayerController : MonoBehaviour
{
    public float walkSpeed = 6f, sprintSpeed = 9f, jumpHeight = 1.4f, gravity = -24f;
    public Transform view;
    public float lookSensitivity = 2.2f;
    private CharacterController controller;
    private float pitch, verticalVelocity;
    private bool captured = true;
    public bool CreativeMode { get; private set; }
    public int health = 100;
    public int blocks = 24;
    public int selectedSlot;
    public int attackDamage = 25;

    void Start()
    {
        controller = GetComponent<CharacterController>();
        Cursor.lockState = CursorLockMode.Locked;
        Cursor.visible = false;
        if (!view) view = Camera.main.transform;
    }

    void Update()
    {
        if (Input.GetKeyDown(KeyCode.Escape))
        {
            captured = !captured;
            Cursor.lockState = captured ? CursorLockMode.Locked : CursorLockMode.None;
            Cursor.visible = !captured;
        }
        if (captured)
        {
            float mx = Input.GetAxis("Mouse X") * lookSensitivity;
            float my = Input.GetAxis("Mouse Y") * lookSensitivity;
            transform.Rotate(0, mx, 0);
            pitch = Mathf.Clamp(pitch - my, -85, 85);
            view.localRotation = Quaternion.Euler(pitch, 0, 0);
        }

        if (Input.GetKeyDown(KeyCode.E))
        {
            CreativeMode = !CreativeMode;
            if (HUDController.Instance) HUDController.Instance.ShowMessage(CreativeMode ? "CREATIVE MODE — unlimited blocks, no damage" : "SURVIVAL MODE");
        }
        for (int i=0;i<5;i++) if (Input.GetKeyDown(KeyCode.Alpha1+i)) selectedSlot=i;
        if (Input.GetKeyDown(KeyCode.C))
        {
            if (CreativeMode) HUDController.Instance?.ShowMessage("Creative: crafting is unnecessary");
            else { blocks += 8; HUDController.Instance?.ShowMessage("Crafted 8 building blocks"); }
        }
        if (Input.GetKeyDown(KeyCode.Space) && controller.isGrounded) verticalVelocity = Mathf.Sqrt(jumpHeight * -2f * gravity);
        if (!controller.isGrounded) verticalVelocity += gravity * Time.deltaTime; else if (verticalVelocity < 0) verticalVelocity = -2f;
        float speed = Input.GetKey(KeyCode.LeftShift) ? sprintSpeed : walkSpeed;
        Vector3 move = transform.right * Input.GetAxisRaw("Horizontal") + transform.forward * Input.GetAxisRaw("Vertical");
        if (move.sqrMagnitude > 1) move.Normalize();
        controller.Move((move * speed + Vector3.up * verticalVelocity) * Time.deltaTime);

        if (Input.GetMouseButtonDown(0)) MineOrAttack();
        if (Input.GetMouseButtonDown(1)) PlaceBlock();
        if (HUDController.Instance) HUDController.Instance.Refresh(this);
    }

    void MineOrAttack()
    {
        if (!captured) return;
        Ray ray = new Ray(view.position, view.forward);
        if (Physics.Raycast(ray, out RaycastHit hit, 5f))
        {
            EnemyAI enemy = hit.collider.GetComponentInParent<EnemyAI>();
            if (enemy != null) { enemy.TakeDamage(attackDamage); return; }
            BlockMarker block = hit.collider.GetComponent<BlockMarker>();
            if (block != null && block.world != null)
            {
                block.world.RemoveBlock(block);
                if (!CreativeMode) blocks += 1;
                HUDController.Instance?.ShowMessage("Block mined");
            }
        }
    }

    void PlaceBlock()
    {
        if (!captured || (!CreativeMode && blocks <= 0)) return;
        Ray ray = new Ray(view.position, view.forward);
        if (Physics.Raycast(ray, out RaycastHit hit, 5f))
        {
            Vector3 p = hit.point + hit.normal * .51f;
            Vector3Int grid = new Vector3Int(Mathf.RoundToInt(p.x), Mathf.RoundToInt(p.y), Mathf.RoundToInt(p.z));
            if (Vector3.Distance(transform.position, grid) < 2f) return;
            var world = FindObjectOfType<VoxelWorld>();
            if (world != null) world.CreateBlock(grid, world.GetBuildMaterial(selectedSlot));
            if (!CreativeMode) blocks--;
        }
    }

    public void Damage(int amount)
    {
        if (CreativeMode) return;
        health = Mathf.Max(0, health - amount);
        if (health <= 0)
        {
            health = 100;
            transform.position = new Vector3(0, 12, 0);
            HUDController.Instance?.ShowMessage("You were defeated — respawned at the portal hub");
        }
    }
}
""",
"Assets/Scripts/EnemyAI.cs": r"""
using UnityEngine;

public class EnemyAI : MonoBehaviour
{
    public int maxHealth = 60;
    public int damage = 8;
    public float speed = 2.2f;
    public bool boss;
    private int health;
    private Transform target;
    private float nextHit;

    void Start() { health = maxHealth; FindTarget(); }
    void FindTarget()
    {
        var p = FindObjectOfType<PlayerController>();
        if (p) target = p.transform;
    }
    void Update()
    {
        if (!target) { FindTarget(); return; }
        float d = Vector3.Distance(transform.position, target.position);
        if (d < (boss ? 24f : 15f))
        {
            Vector3 direction = target.position - transform.position; direction.y = 0;
            if (direction.sqrMagnitude > 0.1f)
            {
                transform.rotation = Quaternion.Slerp(transform.rotation, Quaternion.LookRotation(direction), Time.deltaTime * 4);
                if (d > 1.8f) transform.position += transform.forward * speed * Time.deltaTime;
            }
            if (d < 2.2f && Time.time > nextHit)
            {
                nextHit = Time.time + 1.5f;
                target.GetComponent<PlayerController>()?.Damage(damage);
            }
        }
    }
    public void TakeDamage(int amount)
    {
        health -= amount;
        if (health <= 0)
        {
            if (boss) RealmManager.Instance?.SetMessage("BOSS DEFEATED! Realm champion!");
            Destroy(gameObject);
        }
        else if (HUDController.Instance) HUDController.Instance.ShowMessage((boss ? "Boss" : "Slime") + " HP: " + health + "/" + maxHealth);
    }
}
""",
"Assets/Scripts/RealmPortal.cs": r"""
using UnityEngine;

public class RealmPortal : MonoBehaviour
{
    public RealmType destination;
    private Transform player;
    void Start() { var p = FindObjectOfType<PlayerController>(); if (p) player = p.transform; }
    void Update()
    {
        if (!player) { var p = FindObjectOfType<PlayerController>(); if(p) player=p.transform; return; }
        if (Vector3.Distance(player.position, transform.position) < 3f && Input.GetKeyDown(KeyCode.F))
            RealmManager.Instance?.SwitchRealm(destination);
    }
    void OnGUI()
    {
        if (player && Vector3.Distance(player.position, transform.position) < 3f)
        {
            GUI.Box(new Rect(Screen.width/2-115, Screen.height-110, 230, 30), "Press F: Enter " + destination);
        }
    }
}
""",
"Assets/Scripts/HUDController.cs": r"""
using UnityEngine;

public class HUDController : MonoBehaviour
{
    public static HUDController Instance { get; private set; }
    private string message = "";
    private float messageUntil;
    private GUIStyle label, title, center;
    void Awake() { Instance = this; }
    public void ShowMessage(string text) { message = text; messageUntil = Time.time + 3f; }
    public void Refresh(PlayerController p) { }
    void OnGUI()
    {
        if (label == null)
        {
            label = new GUIStyle(GUI.skin.label) { fontSize = 16, normal = { textColor = Color.white } };
            title = new GUIStyle(label) { fontSize = 22, fontStyle = FontStyle.Bold, alignment = TextAnchor.MiddleCenter };
            center = new GUIStyle(label) { alignment = TextAnchor.MiddleCenter, fontSize = 20, fontStyle = FontStyle.Bold };
        }
        var p = FindObjectOfType<PlayerController>();
        GUI.Box(new Rect(12,12,290,132), "");
        GUI.Label(new Rect(24,18,270,28), "FOUR REALMS  //  VOXEL SURVIVAL", label);
        if (p)
        {
            GUI.Label(new Rect(24,45,260,24), "Mode: " + (p.CreativeMode ? "CREATIVE" : "SURVIVAL"), label);
            GUI.Label(new Rect(24,69,260,24), "Health: " + (p.CreativeMode ? "∞" : p.health + "/100"), label);
            GUI.Label(new Rect(24,93,260,24), "Blocks: " + (p.CreativeMode ? "∞" : p.blocks), label);
            GUI.Label(new Rect(24,117,260,24), "Hotbar slot: " + (p.selectedSlot+1) + "   [1–5]", label);
        }
        GUI.Label(new Rect(Screen.width/2-12, Screen.height/2-12,24,24), "+", center);
        GUI.Label(new Rect(12, Screen.height-84, 660, 64), "WASD move • Mouse look • Space jump • Shift sprint\nLMB mine/attack • RMB place • E mode • C craft • F portal • Esc cursor", label);
        if (Time.time < messageUntil) GUI.Label(new Rect(Screen.width/2-260, 48, 520, 36), message, title);
    }
}
""",
"Assets/Editor/FourRealmsBootstrap.cs": r"""
#if UNITY_EDITOR
using UnityEditor;
using UnityEditor.SceneManagement;
using UnityEngine;
using UnityEngine.SceneManagement;
using System.IO;

public static class FourRealmsBootstrap
{
    [MenuItem("Four Realms/Create Playable Demo Scene")]
    public static void CreateScene()
    {
        if (!EditorSceneManager.SaveCurrentModifiedScenesIfUserWantsTo()) return;
        var scene = EditorSceneManager.NewScene(NewSceneSetup.EmptyScene, NewSceneMode.Single);
        var manager = new GameObject("Realm Manager").AddComponent<RealmManager>();
        var world = new GameObject("Voxel World").AddComponent<VoxelWorld>();
        SetupMaterials(world);
        manager.world = world;

        var ground = new GameObject("World Root");
        world.transform.SetParent(ground.transform, false);
        world.Generate(RealmType.Overworld);

        var player = new GameObject("Player");
        player.transform.position = new Vector3(0, 12, 0);
        var cc = player.AddComponent<CharacterController>();
        cc.height = 1.8f; cc.radius = .35f; cc.center = new Vector3(0,.9f,0);
        var camObj = new GameObject("Player Camera");
        camObj.tag = "MainCamera";
        camObj.transform.SetParent(player.transform, false);
        camObj.transform.localPosition = new Vector3(0,1.55f,0);
        camObj.AddComponent<Camera>().fieldOfView = 78;
        camObj.AddComponent<AudioListener>();
        player.AddComponent<PlayerController>().view = camObj.transform;
        manager.player = player.transform;

        var light = new GameObject("Sun").AddComponent<Light>();
        light.type = LightType.Directional; light.intensity = 1.15f;
        light.transform.rotation = Quaternion.Euler(48,-32,0);
        RenderSettings.ambientLight = new Color(.55f,.58f,.62f);
        RenderSettings.fog = true; RenderSettings.fogColor = new Color(.58f,.72f,.82f); RenderSettings.fogDensity = .006f;

        MakeVillager(new Vector3(-5, 7, 7));
        MakeVillager(new Vector3(11, 7, -8));
        MakeEnemy(new Vector3(5, 7, 4), false, new Color(.25f,.85f,.3f));
        MakeEnemy(new Vector3(13, 7, 11), false, new Color(.25f,.85f,.3f));
        MakeEnemy(new Vector3(0, 7, 17), true, new Color(.7f,.12f,.85f));

        MakePortal(new Vector3(-12, 7, 0), RealmType.Nether, new Color(1f,.15f,.05f));
        MakePortal(new Vector3(12, 7, 0), RealmType.Aether, new Color(.15f,.8f,1f));
        MakePortal(new Vector3(0, 7, -13), RealmType.End, new Color(.65f,.2f,1f));
        MakePortal(new Vector3(0, 7, 13), RealmType.Overworld, new Color(.25f,1f,.4f));

        new GameObject("HUD").AddComponent<HUDController>();

        string folder = "Assets/Scenes";
        if (!Directory.Exists(folder)) Directory.CreateDirectory(folder);
        string path = folder + "/FourRealmsDemo.unity";
        EditorSceneManager.SaveScene(scene, path);
        AssetDatabase.SaveAssets();
        AssetDatabase.Refresh();
        Selection.activeObject = AssetDatabase.LoadAssetAtPath<SceneAsset>(path);
        EditorGUIUtility.PingObject(Selection.activeObject);
        Debug.Log("Four Realms demo created at " + path + ". Press Play to begin.");
    }

    static void SetupMaterials(VoxelWorld w)
    {
        w.grass = Mat("Grass", new Color(.25f,.62f,.24f));
        w.dirt = Mat("Dirt", new Color(.43f,.28f,.16f));
        w.stone = Mat("Stone", new Color(.42f,.45f,.49f));
        w.sand = Mat("Sand", new Color(.78f,.69f,.43f));
        w.nether = Mat("Nether", new Color(.58f,.16f,.12f));
        w.aether = Mat("Aether", new Color(.45f,.8f,.94f));
        w.endStone = Mat("End Stone", new Color(.65f,.59f,.75f));
        w.wood = Mat("Wood", new Color(.35f,.19f,.08f));
        w.leaves = Mat("Leaves", new Color(.12f,.42f,.16f));
    }
    static Material Mat(string name, Color color)
    {
        var shader = Shader.Find("Universal Render Pipeline/Lit");
        if (shader == null) shader = Shader.Find("Standard");
        var mat = new Material(shader) { name = name, color = color };
        return mat;
    }
    static void MakeVillager(Vector3 pos)
    {
        var villager = GameObject.CreatePrimitive(PrimitiveType.Capsule);
        villager.name = "Villager"; villager.transform.position = pos;
        villager.GetComponent<Renderer>().sharedMaterial = Mat("Villager Tunic", new Color(.55f,.35f,.2f));
        var head = GameObject.CreatePrimitive(PrimitiveType.Cube);
        head.name = "Villager Head"; head.transform.SetParent(villager.transform);
        head.transform.localPosition = new Vector3(0,.65f,0); head.transform.localScale = new Vector3(.65f,.65f,.65f);
        head.GetComponent<Renderer>().sharedMaterial = Mat("Villager Skin", new Color(.82f,.62f,.42f));
    }
    static void MakeEnemy(Vector3 pos, bool boss, Color color)
    {
        var enemy = GameObject.CreatePrimitive(boss ? PrimitiveType.Cube : PrimitiveType.Sphere);
        enemy.name = boss ? "Realm Boss" : "Hostile Slime";
        enemy.transform.position = pos;
        enemy.transform.localScale = boss ? new Vector3(2.5f,3f,2.5f) : new Vector3(1.25f,1.1f,1.25f);
        enemy.GetComponent<Renderer>().sharedMaterial = Mat(boss ? "Boss Material" : "Slime Material", color);
        var ai = enemy.AddComponent<EnemyAI>(); ai.boss = boss; ai.maxHealth = boss ? 400 : 60; ai.damage = boss ? 20 : 8; ai.speed = boss ? 1.5f : 2.2f;
    }
    static void MakePortal(Vector3 pos, RealmType destination, Color color)
    {
        var portal = GameObject.CreatePrimitive(PrimitiveType.Cube);
        portal.name = destination + " Portal"; portal.transform.position = pos;
        portal.transform.localScale = new Vector3(2.3f,4f,.5f);
        portal.GetComponent<Renderer>().sharedMaterial = Mat(destination + " Portal Glow", color);
        portal.AddComponent<RealmPortal>().destination = destination;
    }
}
#endif
"""
}

# Write project files and zip.
os.makedirs(root, exist_ok=True)
for rel, content in files.items():
    full = os.path.join(root, rel)
    os.makedirs(os.path.dirname(full), exist_ok=True)
    with open(full, "w", encoding="utf-8", newline="\n") as f:
        f.write(textwrap.dedent(content).lstrip())

zip_path = "/mnt/data/FourRealms_Unity_Starter.zip"
with zipfile.ZipFile(zip_path, "w", zipfile.ZIP_DEFLATED) as z:
    for base, dirs, filenames in os.walk(root):
        for filename in filenames:
            full = os.path.join(base, filename)
            z.write(full, os.path.relpath(full, "/mnt/data"))

print("Created Unity starter ZIP:", zip_path)
print("Included files:")
for rel in files:
    print(" -", rel)
