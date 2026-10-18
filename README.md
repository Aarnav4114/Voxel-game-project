import os, zipfile, json, textwrap

root = "/mnt/data/FourDimensionVoxelSurvival"
os.makedirs(root + "/Assets/Scripts", exist_ok=True)
os.makedirs(root + "/Assets/Editor", exist_ok=True)
os.makedirs(root + "/Packages", exist_ok=True)
os.makedirs(root + "/ProjectSettings", exist_ok=True)

files = {
"Assets/Scripts/VoxelGameBootstrap.cs": r'''
using UnityEngine;
using System.Collections.Generic;

public class VoxelGameBootstrap : MonoBehaviour
{
    public static VoxelGameBootstrap I;
    public enum Dimension { Overworld, Nether, Aether, End }
    public Dimension CurrentDimension { get; private set; } = Dimension.Overworld;
    public bool CreativeMode { get; private set; }
    public int Health { get; private set; } = 100;
    public int Wood { get; private set; } = 12;
    public int Stone { get; private set; } = 24;
    public int Crystal { get; private set; } = 0;
    public int Food { get; private set; } = 5;
    public string Status = "Explore, gather, build and survive.";
    public Material grassMat, dirtMat, stoneMat, woodMat, leafMat, lavaMat, crystalMat, portalMat, obsidianMat, sandMat;
    readonly List<GameObject> generated = new List<GameObject>();
    readonly List<GameObject> mobs = new List<GameObject>();

    void Awake() { I = this; }
    void Start()
    {
        Application.targetFrameRate = 90;
        CreateMaterials();
        SetupLighting();
        if (FindObjectOfType<VoxelPlayerController>() == null) CreatePlayer();
        if (FindObjectOfType<VoxelHUD>() == null) new GameObject("Voxel HUD").AddComponent<VoxelHUD>();
        BuildDimension(Dimension.Overworld);
    }

    Material MakeMat(string name, Color color, bool emission = false)
    {
        Shader shader = Shader.Find("Universal Render Pipeline/Lit");
        if (shader == null) shader = Shader.Find("Standard");
        Material m = new Material(shader) { name = name, color = color };
        if (emission) { m.EnableKeyword("_EMISSION"); m.SetColor("_EmissionColor", color * 1.6f); }
        return m;
    }
    void CreateMaterials()
    {
        grassMat = MakeMat("Grass", new Color(.20f,.48f,.19f));
        dirtMat = MakeMat("Dirt", new Color(.34f,.22f,.13f));
        stoneMat = MakeMat("Stone", new Color(.43f,.46f,.49f));
        woodMat = MakeMat("Wood", new Color(.42f,.25f,.11f));
        leafMat = MakeMat("Leaves", new Color(.10f,.36f,.12f));
        lavaMat = MakeMat("Lava Glow", new Color(1f,.18f,.02f), true);
        crystalMat = MakeMat("Aether Crystal", new Color(.12f,.82f,1f), true);
        portalMat = MakeMat("Portal Glow", new Color(.60f,.12f,1f), true);
        obsidianMat = MakeMat("Obsidian", new Color(.08f,.055f,.15f));
        sandMat = MakeMat("Ash Sand", new Color(.65f,.54f,.40f));
    }
    void SetupLighting()
    {
        RenderSettings.ambientLight = new Color(.52f,.56f,.62f);
        Light light = FindObjectOfType<Light>();
        if (!light) { GameObject sun = new GameObject("Sunlight"); light = sun.AddComponent<Light>(); light.type = LightType.Directional; sun.transform.rotation = Quaternion.Euler(48,-30,0); }
        light.intensity = 1.2f;
    }
    void CreatePlayer()
    {
        GameObject p = new GameObject("Player");
        p.transform.position = new Vector3(0, 5, -4);
        p.AddComponent<CharacterController>();
        VoxelPlayerController controller = p.AddComponent<VoxelPlayerController>();
        GameObject cameraObject = new GameObject("Player Camera");
        cameraObject.transform.SetParent(p.transform);
        cameraObject.transform.localPosition = new Vector3(0, 1.55f, 0);
        Camera cam = cameraObject.AddComponent<Camera>();
        cam.fieldOfView = 75;
        cam.tag = "MainCamera";
        if (!FindObjectOfType<AudioListener>()) cameraObject.AddComponent<AudioListener>();
        controller.view = cameraObject.transform;
    }
    GameObject Cube(string name, Vector3 pos, Vector3 scale, Material mat)
    {
        GameObject g = GameObject.CreatePrimitive(PrimitiveType.Cube);
        g.name = name; g.transform.position = pos; g.transform.localScale = scale;
        if (mat) g.GetComponent<Renderer>().sharedMaterial = mat;
        generated.Add(g);
        return g;
    }
    void BuildDimension(Dimension d)
    {
        CurrentDimension = d;
        foreach (GameObject g in generated) if (g) Destroy(g);
        generated.Clear();
        foreach (GameObject m in mobs) if (m) Destroy(m);
        mobs.Clear();

        RenderSettings.fog = true;
        RenderSettings.fogColor = SkyColor(d);
        RenderSettings.fogDensity = d == Dimension.Nether ? .018f : .004f;
        RenderSettings.ambientLight = d == Dimension.Nether ? new Color(.42f,.16f,.11f) : new Color(.52f,.56f,.62f);
        if (Camera.main) Camera.main.backgroundColor = SkyColor(d);

        BuildTerrain(d);
        if (d == Dimension.Overworld)
        {
            BuildVillage();
            BuildTree(6,0,6); BuildTree(-8,0,7); BuildTree(9,0,-8); BuildTree(-12,0,-5);
            BuildPatch(10,-8,5,sandMat);
        }
        else if (d == Dimension.Nether)
        {
            for (int i=0;i<12;i++) Cube("Lava Pool", new Vector3(Random.Range(-18,18), .08f, Random.Range(-18,18)), new Vector3(Random.Range(2,5),.15f,Random.Range(2,5)), lavaMat);
            BuildFortress();
        }
        else if (d == Dimension.Aether)
        {
            for (int i=0;i<8;i++)
            {
                Vector3 c = new Vector3(Random.Range(-18,18), Random.Range(6,12), Random.Range(-18,18));
                for (int x=-2;x<=2;x++) for (int z=-2;z<=2;z++) Cube("Floating Island", c+new Vector3(x,0,z), Vector3.one, i%2==0?grassMat:stoneMat);
                Cube("Aether Crystal", c+Vector3.up, Vector3.one*.65f, crystalMat);
            }
        }
        else
        {
            for (int i=0;i<8;i++)
            {
                Vector3 c = new Vector3(Random.Range(-18,18),0,Random.Range(-18,18));
                Cube("Obsidian Tower", c+new Vector3(0,Random.Range(3,9),0), new Vector3(2,Random.Range(6,18),2), obsidianMat);
            }
        }
        CreatePortal();
        SpawnMobs(d);
        if (d == Dimension.End) SpawnBoss("Void Dragon", new Color(.28f,.08f,.42f), 300);
        else if (d == Dimension.Nether) SpawnBoss("Infernal Brute", new Color(.85f,.12f,.035f), 220);
        else if (d == Dimension.Aether) SpawnBoss("Storm Guardian", new Color(.08f,.65f,.95f), 200);

        VoxelPlayerController player = FindObjectOfType<VoxelPlayerController>();
        if (player) { player.transform.position = new Vector3(0, d == Dimension.Aether ? 14 : 4, -4); player.ResetMotion(); }
        Status = d + " loaded. Explore and find the glowing portal.";
    }
    Color SkyColor(Dimension d)
    {
        switch (d)
        {
            case Dimension.Nether: return new Color(.22f,.055f,.035f);
            case Dimension.Aether: return new Color(.30f,.68f,.95f);
            case Dimension.End: return new Color(.035f,.015f,.07f);
            default: return new Color(.48f,.72f,.95f);
        }
    }
    void BuildTerrain(Dimension d)
    {
        for (int x=-22;x<=22;x++) for (int z=-22;z<=22;z++)
        {
            float h = d == Dimension.Overworld ? Mathf.PerlinNoise((x+50)*.09f,(z+50)*.09f)*2.1f : 0;
            Material top = d==Dimension.Nether ? obsidianMat : d==Dimension.End ? obsidianMat : d==Dimension.Aether ? crystalMat : grassMat;
            if (d==Dimension.Nether && Mathf.PerlinNoise(x*.2f,z*.2f)>.76f) top=stoneMat;
            Cube("Voxel Block", new Vector3(x, Mathf.Floor(h), z), Vector3.one, top);
            Cube("Subsoil Block", new Vector3(x, Mathf.Floor(h)-1, z), Vector3.one, d==Dimension.Overworld?dirtMat:stoneMat);
            if (d==Dimension.Overworld && (x*x+z*z)>150 && Random.value<.012f)
                Cube("Crystal Ore", new Vector3(x,Mathf.Floor(h)+.55f,z),Vector3.one*.42f,crystalMat);
        }
    }
    void BuildTree(int x,int y,int z)
    {
        int h=Random.Range(4,7);
        for(int i=1;i<=h;i++) Cube("Tree Trunk",new Vector3(x,y+i,z),Vector3.one,woodMat);
        for(int yy=h-1;yy<=h+1;yy++) for(int xx=-2;xx<=2;xx++) for(int zz=-2;zz<=2;zz++)
            if(Mathf.Abs(xx)+Mathf.Abs(zz)+(yy==h?0:1)<5) Cube("Leaf Block",new Vector3(x+xx,y+yy,z+zz),Vector3.one,leafMat);
    }
    void BuildPatch(int x,int z,int size,Material mat)
    {
        for(int xx=0;xx<size;xx++) for(int zz=0;zz<size;zz++) Cube("Biome Patch",new Vector3(x+xx,.55f,z+zz),Vector3.one,mat);
    }
    void BuildVillage()
    {
        for(int i=0;i<4;i++)
        {
            int x=-12+i*6, z=8+(i%2)*4;
            for(int xx=0;xx<4;xx++) for(int zz=0;zz<4;zz++) Cube("Village Floor",new Vector3(x+xx,1,z+zz),Vector3.one,woodMat);
            for(int yy=2;yy<5;yy++) for(int xx=0;xx<4;xx++) for(int zz=0;zz<4;zz++)
                if(xx==0||xx==3||zz==0||zz==3) Cube("Village Wall",new Vector3(x+xx,yy,z+zz),Vector3.one,yy==4?woodMat:dirtMat);
            for(int xx=-1;xx<5;xx++) for(int zz=-1;zz<5;zz++) Cube("Village Roof",new Vector3(x+xx,5,z+zz),Vector3.one,obsidianMat);
            GameObject villager=GameObject.CreatePrimitive(PrimitiveType.Capsule);
            villager.name="Villager Trader"; villager.transform.position=new Vector3(x+2,2,z+2); villager.transform.localScale=new Vector3(.7f,1.15f,.7f);
            villager.GetComponent<Renderer>().material=MakeMat("Villager Tunic",new Color(.55f,.32f,.18f));
            generated.Add(villager);
        }
    }
    void BuildFortress()
    {
        for(int x=-5;x<=5;x++) for(int z=8;z<=12;z++) Cube("Nether Fortress Floor",new Vector3(x,2,z),Vector3.one,obsidianMat);
        for(int x=-5;x<=5;x++) for(int y=2;y<=6;y++)
        {
            Cube("Fortress Wall",new Vector3(x,y,8),Vector3.one,obsidianMat);
            Cube("Fortress Wall",new Vector3(x,y,12),Vector3.one,obsidianMat);
        }
    }
    void CreatePortal()
    {
        GameObject portal=Cube("Dimension Portal",new Vector3(0,2,17),new Vector3(3,4,.45f),portalMat);
        portal.GetComponent<BoxCollider>().isTrigger=true;
        Rigidbody rb=portal.AddComponent<Rigidbody>(); rb.isKinematic=true; rb.useGravity=false;
        portal.AddComponent<PortalTrigger>();
    }
    void SpawnMobs(Dimension d)
    {
        int count=d==Dimension.Overworld?5:7;
        for(int i=0;i<count;i++)
        {
            GameObject mob=GameObject.CreatePrimitive(PrimitiveType.Cube);
            mob.name=d==Dimension.Overworld?(i%2==0?"Hostile Zombie":"Jungle Spider"):d==Dimension.Nether?"Magma Fiend":d==Dimension.Aether?"Sky Wraith":"End Stalker";
            mob.transform.position=new Vector3(Random.Range(-16,16),2,Random.Range(1,20));
            mob.transform.localScale=new Vector3(1,1.6f,1);
            Color c=d==Dimension.Nether?new Color(.9f,.18f,.035f):d==Dimension.Aether?new Color(.12f,.65f,.95f):d==Dimension.End?new Color(.4f,.1f,.55f):new Color(.25f,.38f,.18f);
            mob.GetComponent<Renderer>().material=MakeMat(mob.name,c,d!=Dimension.Overworld);
            VoxelMob ai=mob.AddComponent<VoxelMob>(); ai.damage=CreativeMode?0:8;
            mobs.Add(mob);
        }
    }
    void SpawnBoss(string name,Color color,int hp)
    {
        GameObject boss=GameObject.CreatePrimitive(PrimitiveType.Cube);
        boss.name=name+" BOSS"; boss.transform.position=new Vector3(0,3,11); boss.transform.localScale=new Vector3(3.5f,4,3.5f);
        boss.GetComponent<Renderer>().material=MakeMat(name,color,true);
        boss.AddComponent<VoxelBoss>().health=hp;
    }
    public void ChangeDimension(){BuildDimension((Dimension)(((int)CurrentDimension+1)%4));}
    public void ToggleMode(){CreativeMode=!CreativeMode;Status=CreativeMode?"Creative mode: unlimited building, no damage.":"Survival mode: gather resources and watch health.";}
    public void DamagePlayer(int amount)
    {
        if(CreativeMode)return;
        Health=Mathf.Max(0,Health-amount);
        if(Health<=0){Health=100;VoxelPlayerController p=FindObjectOfType<VoxelPlayerController>();if(p)p.transform.position=new Vector3(0,5,-4);Status="You respawned. Keep exploring!";}
    }
    public void Heal(int amount){Health=Mathf.Min(100,Health+amount);}
    public void Gather(string kind)
    {
        if(kind=="wood")Wood+=3;else if(kind=="stone")Stone+=3;else if(kind=="crystal")Crystal+=1;
        Status="Gathered "+kind+". Wood "+Wood+" | Stone "+Stone+" | Crystal "+Crystal;
    }
    public bool SpendBuildBlock()
    {
        if(CreativeMode)return true;
        if(Stone>0){Stone--;return true;}
        Status="Out of stone. Press C to craft if you have crystal.";return false;
    }
    public void Craft()
    {
        if(Crystal>0){Crystal--;Stone+=3;Status="Crafted 3 stone blocks.";}
        else if(Wood>=3){Wood-=3;Food++;Status="Crafted a ration (+1 food).";}
        else Status="Need 1 crystal or 3 wood to craft.";
    }
    public void Eat()
    {
        if(Food>0){Food--;Heal(25);Status="Ate food (+25 health).";}else Status="No food left. Gather resources.";
    }
}
''',
"Assets/Scripts/VoxelPlayerController.cs": r'''
using UnityEngine;
[RequireComponent(typeof(CharacterController))]
public class VoxelPlayerController : MonoBehaviour
{
    public Transform view;
    public float speed=6f, sprintSpeed=9f, jumpHeight=1.4f, mouseSensitivity=2.2f;
    CharacterController cc; float verticalVelocity, pitch, attackCooldown;
    void Start(){cc=GetComponent<CharacterController>();Cursor.lockState=CursorLockMode.Locked;Cursor.visible=false;}
    void Update()
    {
        if(!cc)cc=GetComponent<CharacterController>();
        if(Input.GetKeyDown(KeyCode.Escape)){Cursor.lockState=CursorLockMode.None;Cursor.visible=true;}
        if(Cursor.lockState==CursorLockMode.None&&Input.GetMouseButtonDown(0)){Cursor.lockState=CursorLockMode.Locked;Cursor.visible=false;}
        if(Cursor.lockState==CursorLockMode.Locked)
        {
            transform.Rotate(0,Input.GetAxis("Mouse X")*mouseSensitivity,0);
            pitch=Mathf.Clamp(pitch-Input.GetAxis("Mouse Y")*mouseSensitivity,-85,85);
            if(view)view.localRotation=Quaternion.Euler(pitch,0,0);
        }
        Vector3 move=transform.right*Input.GetAxis("Horizontal")+transform.forward*Input.GetAxis("Vertical");
        float s=Input.GetKey(KeyCode.LeftShift)?sprintSpeed:speed;
        if(cc.isGrounded&&verticalVelocity<0)verticalVelocity=-2f;
        if(Input.GetButtonDown("Jump")&&cc.isGrounded)verticalVelocity=Mathf.Sqrt(jumpHeight*-2f*Physics.gravity.y);
        verticalVelocity+=Physics.gravity.y*Time.deltaTime;
        cc.Move((move*s+Vector3.up*verticalVelocity)*Time.deltaTime);
        attackCooldown-=Time.deltaTime;
        if(Input.GetMouseButtonDown(0)&&attackCooldown<=0){Attack();attackCooldown=.35f;}
        if(Input.GetMouseButtonDown(1))PlaceBlock();
        if(Input.GetKeyDown(KeyCode.Alpha1))VoxelGameBootstrap.I.Gather("wood");
        if(Input.GetKeyDown(KeyCode.Alpha2))VoxelGameBootstrap.I.Gather("stone");
        if(Input.GetKeyDown(KeyCode.Alpha3))VoxelGameBootstrap.I.Gather("crystal");
    }
    void Attack()
    {
        if(!view)return;
        if(Physics.Raycast(new Ray(view.position,view.forward),out RaycastHit hit,4.5f))
        {
            VoxelMob mob=hit.collider.GetComponent<VoxelMob>();if(mob){mob.TakeDamage(25);return;}
            VoxelBoss boss=hit.collider.GetComponent<VoxelBoss>();if(boss){boss.TakeDamage(20);return;}
            string n=hit.collider.gameObject.name;
            if(n.Contains("Ore")||n.Contains("Crystal"))VoxelGameBootstrap.I.Gather("crystal");
            else if(n.Contains("Tree")||n.Contains("Leaf"))VoxelGameBootstrap.I.Gather("wood");
            else if(n.Contains("Voxel")||n.Contains("Subsoil"))VoxelGameBootstrap.I.Gather("stone");
        }
    }
    void PlaceBlock()
    {
        if(!view||!VoxelGameBootstrap.I.SpendBuildBlock())return;
        if(Physics.Raycast(new Ray(view.position,view.forward),out RaycastHit hit,6f))
        {
            Vector3 p=hit.point+hit.normal*.51f;p=new Vector3(Mathf.Round(p.x),Mathf.Round(p.y),Mathf.Round(p.z));
            GameObject g=GameObject.CreatePrimitive(PrimitiveType.Cube);g.name="Player Placed Block";g.transform.position=p;
            g.GetComponent<Renderer>().material=VoxelGameBootstrap.I.stoneMat;
        }
    }
    public void ResetMotion(){verticalVelocity=0;if(cc){cc.enabled=false;cc.enabled=true;}}
}
''',
"Assets/Scripts/VoxelMob.cs": r'''
using UnityEngine;
public class VoxelMob : MonoBehaviour
{
    public int health=50, damage=8;
    Transform player; float attackTimer,wanderTimer; Vector3 wander;
    void Start(){VoxelPlayerController p=FindObjectOfType<VoxelPlayerController>();if(p)player=p.transform;wander=Random.insideUnitSphere*5;}
    void Update()
    {
        if(!player)return;
        float d=Vector3.Distance(transform.position,player.position);
        if(d<14)
        {
            Vector3 dir=player.position-transform.position;dir.y=0;
            if(dir.sqrMagnitude>.1f)transform.rotation=Quaternion.LookRotation(dir);
            if(d>1.8f)transform.position+=transform.forward*2.1f*Time.deltaTime;
            else{attackTimer-=Time.deltaTime;if(attackTimer<=0){VoxelGameBootstrap.I.DamagePlayer(damage);attackTimer=1.4f;}}
        }
        else
        {
            wanderTimer-=Time.deltaTime;
            if(wanderTimer<=0){wander=Random.insideUnitSphere*7;wander.y=0;wanderTimer=3;}
            transform.position=Vector3.MoveTowards(transform.position,player.position+wander,Time.deltaTime*.5f);
        }
    }
    public void TakeDamage(int amount){health-=amount;if(health<=0){VoxelGameBootstrap.I.Gather("crystal");Destroy(gameObject);}}
}
''',
"Assets/Scripts/VoxelBoss.cs": r'''
using UnityEngine;
public class VoxelBoss : MonoBehaviour
{
    public int health=250;
    Transform player; float timer;
    void Start(){VoxelPlayerController p=FindObjectOfType<VoxelPlayerController>();if(p)player=p.transform;}
    void Update()
    {
        if(!player)return;
        Vector3 dir=player.position-transform.position;dir.y=0;
        if(dir.sqrMagnitude>.1f)transform.rotation=Quaternion.LookRotation(dir);
        if(Vector3.Distance(player.position,transform.position)>3)transform.position+=transform.forward*1.2f*Time.deltaTime;
        timer-=Time.deltaTime;
        if(timer<=0&&Vector3.Distance(player.position,transform.position)<5){VoxelGameBootstrap.I.DamagePlayer(15);timer=2.2f;}
    }
    public void TakeDamage(int amount){health-=amount;if(health<=0){VoxelGameBootstrap.I.Gather("crystal");VoxelGameBootstrap.I.Gather("stone");Destroy(gameObject);}}
}
''',
"Assets/Scripts/PortalTrigger.cs": r'''
using UnityEngine;
public class PortalTrigger : MonoBehaviour
{
    bool cooldown;
    void OnTriggerEnter(Collider other)
    {
        if(!cooldown&&other.GetComponent<VoxelPlayerController>())
        {
            cooldown=true;VoxelGameBootstrap.I.ChangeDimension();Invoke(nameof(ResetCooldown),1.5f);
        }
    }
    void ResetCooldown(){cooldown=false;}
}
''',
"Assets/Scripts/VoxelHUD.cs": r'''
using UnityEngine;
public class VoxelHUD : MonoBehaviour
{
    GUIStyle title,body,small; Texture2D panel;
    void Start(){panel=new Texture2D(1,1);panel.SetPixel(0,0,new Color(.025f,.045f,.07f,.88f));panel.Apply();}
    void OnGUI()
    {
        if(title==null)
        {
            title=new GUIStyle(GUI.skin.label);title.fontSize=21;title.fontStyle=FontStyle.Bold;title.normal.textColor=new Color(.15f,.95f,1f);
            body=new GUIStyle(GUI.skin.label);body.fontSize=14;body.normal.textColor=Color.white;
            small=new GUIStyle(GUI.skin.label);small.fontSize=12;small.normal.textColor=new Color(.75f,.85f,.9f);
        }
        VoxelGameBootstrap g=VoxelGameBootstrap.I;if(!g)return;
        GUI.DrawTexture(new Rect(14,14,365,145),panel);
        GUI.Label(new Rect(26,19,340,28),"VOXEL REALMS // SURVIVAL",title);
        GUI.Label(new Rect(26,52,340,21),"Dimension: "+g.CurrentDimension+" | Mode: "+(g.CreativeMode?"Creative":"Survival"),body);
        GUI.Label(new Rect(26,76,340,21),"Health "+g.Health+"/100 | Food "+g.Food,body);
        GUI.Label(new Rect(26,100,340,21),"Wood "+g.Wood+" Stone "+g.Stone+" Crystal "+g.Crystal,body);
        GUI.Label(new Rect(26,126,340,21),g.Status,small);
        float w=Mathf.Min(Screen.width-28,800);
        GUI.DrawTexture(new Rect(14,Screen.height-91,w,78),panel);
        GUI.Label(new Rect(26,Screen.height-85,w-24,20),"WASD move • Mouse look • Space jump • Shift sprint • Left click attack/mine • Right click build",body);
        GUI.Label(new Rect(26,Screen.height-62,w-24,20),"F cycle dimensions • M creative/survival • C craft • E eat • 1 wood  2 stone  3 crystal • Esc unlock mouse",small);
        GUI.Label(new Rect(26,Screen.height-40,w-24,20),"Portal is ahead at Z=17. Travel: Overworld → Nether → Aether → End.",small);
        GUI.Label(new Rect(Screen.width/2-5,Screen.height/2-12,20,24),"+",title);
        if(Cursor.lockState==CursorLockMode.None)GUI.Label(new Rect(Screen.width/2-100,Screen.height/2+18,220,30),"CLICK TO RESUME",title);
    }
}
''',
"Assets/Editor/BootstrapSceneCreator.cs": r'''
#if UNITY_EDITOR
using UnityEditor;
using UnityEditor.SceneManagement;
using UnityEngine;
public static class BootstrapSceneCreator
{
    [MenuItem("Voxel Realms/Create Playable Scene")]
    public static void Create()
    {
        var scene=EditorSceneManager.NewScene(NewSceneSetup.EmptyScene,NewSceneMode.Single);
        var bootstrap=new GameObject("Voxel Game Bootstrap");
        bootstrap.AddComponent<VoxelGameBootstrap>();
        var sun=new GameObject("Sunlight");
        var light=sun.AddComponent<Light>();
        light.type=LightType.Directional;light.intensity=1.2f;sun.transform.rotation=Quaternion.Euler(48,-30,0);
        EditorSceneManager.SaveScene(scene,"Assets/VoxelSurvival.unity");
        Selection.activeGameObject=bootstrap;
        Debug.Log("Voxel Realms scene created at Assets/VoxelSurvival.unity. Press Play.");
    }
    [InitializeOnLoadMethod]
    static void CreateSceneIfMissing()
    {
        EditorApplication.delayCall += () =>
        {
            if (!Application.isPlaying && !System.IO.File.Exists("Assets/VoxelSurvival.unity"))
                Create();
        };
    }
}
'''
}

manifest = {
    "dependencies": {
        "com.unity.ide.visualstudio": "2.0.22",
        "com.unity.test-framework": "1.1.33",
        "com.unity.ugui": "1.0.0"
    }
}
files["Packages/manifest.json"] = json.dumps(manifest, indent=2)
files["ProjectSettings/ProjectVersion.txt"] = "m_EditorVersion: 2022.3.45f1\n"
files["README.md"] = """# Four-Dimension Voxel Survival — Unity Starter Project

A playable first-person voxel prototype built with runtime-generated cubes and stylized materials. No external art assets are required.

## Included
- First-person movement, mouse look, jumping and sprinting
- Survival and creative modes
- Four dimensions: Overworld, Nether, Aether and End
- Procedural-looking block terrain, trees, floating islands, lava pools and obsidian towers
- Village buildings and villager NPCs
- Hostile mobs that chase and damage the player
- Boss encounters in the Nether, Aether and End
- Block placement, resource gathering, simple crafting and food healing
- HUD with health, mode, dimension and resources

## Requirements
Unity Hub and Unity Editor 2022.3 LTS or newer. Desktop build target.

## Run
1. Extract the ZIP.
2. Open Unity Hub and choose **Add > Add project from disk**.
3. Select the extracted `FourDimensionVoxelSurvival` folder.
4. Open the project and wait for scripts to compile.
5. Open `Assets/VoxelSurvival.unity`. If it isn't present, use **Voxel Realms > Create Playable Scene** from the Unity menu.
6. Press **Play**.

## Controls
- **W A S D** — move
- **Mouse** — look
- **Space** — jump
- **Left Shift** — sprint
- **Left mouse** — attack mobs/bosses or gather blocks in reach
- **Right mouse** — place a stone block
- **F** — cycle dimensions (Overworld → Nether → Aether → End)
- **M** — toggle Survival / Creative
- **C** — craft: crystal into 3 stone, or 3 wood into food
- **E** — eat food and restore health
- **1 / 2 / 3** — prototype resource shortcuts for wood / stone / crystal
- **Esc** — release mouse; click to recapture

The glowing portal is placed ahead of the spawn point. Press **F** to cycle dimensions immediately.

## Notes
This is a first playable prototype, not a finished commercial game. The world uses individual cubes and material colors rather than a chunked voxel engine or texture atlas. It is intentionally self-contained; the next steps for a fuller game would be optimized chunk meshes, persistent worlds, real inventory UI, animation, audio, and more sophisticated combat.
"""
for rel, content in files.items():
    path = os.path.join(root, rel)
    os.makedirs(os.path.dirname(path), exist_ok=True)
    with open(path, "w", encoding="utf-8") as f:
        f.write(content.strip() + "\n")

zip_path = "/mnt/data/FourDimensionVoxelSurvival.zip"
with zipfile.ZipFile(zip_path, "w", zipfile.ZIP_DEFLATED) as archive:
    for folder, _, filenames in os.walk(root):
        for filename in filenames:
            full = os.path.join(folder, filename)
            archive.write(full, os.path.relpath(full, root))

print(f"Created Unity starter project ZIP: {zip_path}")
print(f"Included {len(files)} files; archive size: {os.path.getsize(zip_path):,} bytes")
