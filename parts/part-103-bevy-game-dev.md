# Part 103: Game Development ด้วย Bevy Engine เบื้องต้น

> โมดูล: Specialized Domains | ระดับ: สูง | เวลาโดยประมาณ: 200 นาที

## เป้าหมายของบทเรียน

- อธิบายได้ว่าทำไม game engine ส่วนใหญ่ในยุคปัจจุบันเลือกใช้ Entity Component System (ECS) แทน class
  inheritance แบบ OOP ดั้งเดิม และเหตุผลเชิงโครงสร้างที่ทำให้ ECS เข้ากับปรัชญา composition-over-inheritance
  ของ Rust ได้อย่างเป็นธรรมชาติ
- เข้าใจสามเสาหลักของ ECS ใน Bevy: **Entity** (ID เปล่า ๆ), **Component** (ข้อมูลที่แนบกับ entity ผ่าน
  `#[derive(Component)]`), และ **System** (ฟังก์ชันธรรมดาที่ query หา entity ที่มี component ตามที่ต้องการ)
- ติดตั้งและรัน Bevy app ขั้นต่ำได้จริงด้วย `cargo add bevy` และ `App::new().add_plugins(DefaultPlugins).run()`
  พร้อมเข้าใจปัญหาเรื่องเวลา compile ครั้งแรกที่ยาวนาน และวิธีบรรเทาด้วย `dynamic_linking` feature
- เขียนและอ่าน `Query` ได้หลายรูปแบบ ทั้งแบบ read-only, แบบ mutate, แบบมี tuple ของหลาย component, และแบบกรอง
  ด้วย `With<T>`/`Without<T>`
- ใช้ `Resource` สำหรับข้อมูล global แบบ singleton (เช่น score, game state) แยกออกจาก `Component` ที่ผูกกับ
  entity แต่ละตัว และใช้ระบบ event (ซึ่งในเวอร์ชันปัจจุบันของ Bevy เรียกว่า **Message**) สำหรับการสื่อสาร
  แบบ decoupled ระหว่าง system
- เขียนเกมเล็ก ๆ ที่รวม entity/component/system/resource/message เข้าด้วยกันได้จริง ทดสอบ logic ของเกมแบบ
  headless (ไม่ต้องมีหน้าจอ) และเข้าใจข้อจำกัดของการตรวจสอบแบบนี้ว่าตรวจ "ตรรกะ" ได้ แต่ตรวจ "ภาพที่ render จริง"
  ไม่ได้
- วางตำแหน่ง Bevy เทียบกับตัวเลือกอื่นในโลก game dev ของ Rust ได้อย่างตรงไปตรงมา ทั้งจุดแข็ง (pure Rust,
  performance, community ที่ active) และจุดที่ต้องระวัง (API เปลี่ยนแปลงบ่อยระหว่าง version)

## ความรู้ที่ต้องมีมาก่อน

- **Part 9 (Structs)** และ **Part 10 (Enums and Pattern Matching)**: Component ใน Bevy คือ struct ธรรมดา
  และ Resource บางตัวในบทนี้ก็เป็น enum — ถ้าคุณยังไม่คุ้นกับการนิยาม struct/enum ของตัวเอง ควรกลับไปอ่านสอง
  Part นี้ก่อน
- **Part 19 (Traits Basics)**: แนวคิด "composition over inheritance" ที่ Part 19 อธิบายไว้ (struct ที่ไม่มี
  ความสัมพันธ์กันแต่ implement trait ร่วมกันได้) เป็นรากฐานทางปรัชญาเดียวกับที่ ECS ใช้ในระดับ architecture
  ทั้งเกม
- **Part 25-26 (Iterators)**: `Query` ใน Bevy คืนค่าที่ implement `Iterator` เกือบทุกจุด — `for` loop บน
  `Query`, `.iter()`, `.iter_mut()` ทำงานตามกลไกเดียวกับที่ Part 25-26 อธิบายไว้ทุกประการ
- **Part 37-38 (Threads, Channels)**: ระบบ Message ของ Bevy (หัวข้อ 103.6) เป็นแนวคิด "ส่งข้อความแทนการแชร์
  state" แบบเดียวกับ `mpsc::channel` ใน Part 38 เพียงแต่ทำงานภายใน ECS scheduler แทนที่จะเป็นข้าม OS thread
  ตรง ๆ — และ Bevy เบื้องหลังก็ใช้ thread pool คู่ขนานจริง (`multi_threaded` feature) ซึ่งอาศัยแนวคิดจาก Part 37
- **Part 17 (Packages, Crates, Workspaces)**: เพื่อเข้าใจว่า `bevy` เป็น crate เดียวที่ประกอบขึ้นจาก sub-crate
  เล็ก ๆ กว่า 30 ตัว (`bevy_ecs`, `bevy_app`, `bevy_render`, `bevy_window`, ...) และ feature flag ใน
  `Cargo.toml` เลือกได้ว่าจะรวม sub-crate ไหนเข้ามาบ้าง
- (แนะนำ) **Part 86 (WASM Basics)** ได้เกริ่นไว้แล้วว่า Bevy compile เป็น WASM ได้ — บทนี้ไม่ได้ลงรายละเอียด
  WASM target ซ้ำ แต่จะอ้างอิงกลับไปสั้น ๆ ในหัวข้อสรุป

## เนื้อหา

### 103.1 ทำไมเกมถึงมีปัญหาทาง architecture ที่ต่างจากแอปทั่วไป

ก่อนจะพูดถึง Bevy หรือ ECS เลย ลองตั้งคำถามก่อนว่า "เกม" ต่างจากแอประบบจัดการสินค้าคงคลังหรือระบบธนาคารที่เราเขียน
กันมาตลอดหลักสูตรนี้อย่างไร

ในระบบธุรกิจทั่วไป entity ในระบบ (ลูกค้า, สินค้า, ใบสั่งซื้อ) มักจะ **ค่อนข้างนิ่ง** — โครงสร้างข้อมูลของ
"ลูกค้า" ไม่เปลี่ยนไปมาระหว่างการทำงานของโปรแกรม และจำนวน entity ก็ไม่ได้มากมายถึงหลักหมื่นหลักแสนตัวที่ต้อง
อัปเดตพร้อมกัน 60 ครั้งต่อวินาที

แต่ในเกม สิ่งที่เรียกว่า "ของในเกม" (game object) มีคุณสมบัติที่ต่างออกไปโดยสิ้นเชิง:

1. **มีจำนวนมาก และหลากหลายชนิด** — กระสุน, ศัตรู, ผู้เล่น, item บนพื้น, particle effect, UI element — เกม
   หนึ่งด่านอาจมี object เป็นพันเป็นหมื่นตัวที่ต้องอัปเดตทุกเฟรม
2. **มีชุดคุณสมบัติที่ผสมกันได้หลากหลายรูปแบบ** — ศัตรูตัวหนึ่งอาจจะบินได้แต่ยิงไม่ได้, อีกตัวว่ายน้ำได้และ
   ยิงได้, อีกตัวเดินอย่างเดียวแต่มี boss health bar พิเศษ
3. **ต้องอัปเดตพร้อมกันทุกเฟรม ด้วยความเร็วสูง** — เกมที่ลื่นต้องรันที่ 60 เฟรมต่อวินาที หมายความว่าทุกอย่าง
   ต้องเสร็จภายใน ~16.6 มิลลิวินาที ต่อเฟรม ไม่ว่าจะมี object กี่พันตัวก็ตาม

#### ปัญหาคลาสสิกของ OOP inheritance ในเกม: "Flying Swimming Enemy Problem"

ลองนึกภาพว่าเราออกแบบเกมด้วยแนวคิด OOP แบบดั้งเดิมที่มี class hierarchy คล้ายภาษาอย่าง Java หรือ C++:

```
class Entity {
    x, y: ตำแหน่ง
    render(): แสดงผล
}

class Character extends Entity {
    health: เลือด
    takeDamage(): รับดาเมจ
}

class Enemy extends Character {
    attackPlayer(): โจมตีผู้เล่น
}
```

ทีนี้ทีมออกแบบเกมอยากเพิ่มศัตรูที่ "บินได้" เข้ามา วิธีตาม OOP คือสร้าง `FlyingEnemy extends Enemy` แล้วเพิ่ม
method `fly()` เข้าไป ดูเหมือนสมเหตุสมผลดี

ต่อมาทีมออกแบบอยากเพิ่มศัตรูที่ "ว่ายน้ำได้" เข้ามาอีก ก็สร้าง `SwimmingEnemy extends Enemy` เพิ่ม method
`swim()` — ก็ยังดูโอเค

แล้ววันหนึ่งฝ่ายออกแบบเกมก็มาบอกว่า "อยากได้ศัตรูที่ทั้งบินได้ **และ** ว่ายน้ำได้ — เป็นแมลงปีกที่ดำลงไปใต้น้ำ
ได้ด้วย" คำถามคือ: มันควร extends `FlyingEnemy` หรือ `SwimmingEnemy`?

- ถ้า extends `FlyingEnemy` แล้วเพิ่ม `swim()` เข้าไปเอง ก็ซ้ำโค้ดกับ `SwimmingEnemy`
- ถ้าใช้ multiple inheritance (บางภาษาอนุญาต เช่น C++) ก็เจอปัญหา diamond problem ทันที — ทั้ง
  `FlyingEnemy` และ `SwimmingEnemy` ต่างสืบทอด `Enemy` มา ถ้า `Enemy` มี field ชื่อเดียวกันที่ทั้งสองฝั่งแก้ไข
  คนละแบบ compiler (หรือโปรแกรมเมอร์) ต้องมาตัดสินว่าจะใช้ฝั่งไหน
- ถ้าสร้าง class ใหม่ `FlyingSwimmingEnemy extends Enemy` แยกลำพัง ก็ต้อง copy โค้ดจาก `fly()`/`swim()` มาใหม่
  หรือใช้ pattern ซับซ้อนอย่าง mixin/trait (ในความหมายของภาษาอื่น ไม่ใช่ Rust trait) เพื่อแก้ปัญหานี้ — และยิ่ง
  เกมมีความสามารถ (บิน, ว่ายน้ำ, ล่องหน, ยิงระยะไกล, ระเบิดตัวตอนตาย, ...) มากขึ้นเท่าไหร่ จำนวน class ที่ต้อง
  สร้างเพื่อครอบคลุมทุก **combination** ก็ยิ่งเพิ่มแบบ exponential

นี่คือปัญหาที่วงการเกมรู้จักกันมานานในชื่อ **"Flying Swimming Enemy Problem"** — class inheritance สร้าง
ลำดับชั้นที่ตายตัว (a IS-A b) แต่คุณสมบัติของ object ในเกมจริง ๆ แล้วเป็นเรื่องของ "object นี้ **มี**
ความสามารถอะไรบ้าง" (HAS-A) ซึ่งเป็นการรวมกัน (composition) มากกว่าการสืบทอด (inheritance)

#### ลองแก้ด้วย trait object ของ Rust เอง: ดีขึ้น แต่ยังไม่ใช่คำตอบสุดท้าย

ก่อนจะไปถึง ECS เต็มรูปแบบ นักพัฒนาที่คุ้นกับ Rust อาจคิดว่า "เรามี trait object (`Box<dyn Trait>`) แล้ว
ปัญหา diamond inheritance มันหายไปเองอยู่แล้วไม่ใช่หรือ เพราะ Rust ไม่มี class inheritance ตั้งแต่แรก" — คำ
ตอบคือ **ใช่ครึ่งหนึ่ง** ลองดูตัวอย่างที่ compile และรันได้จริงนี้:

```rust
trait Ability {
    fn describe(&self) -> String;
}

struct FlyAbility { altitude: f32 }
impl Ability for FlyAbility {
    fn describe(&self) -> String { format!("flying at altitude {}", self.altitude) }
}

struct SwimAbility { depth: f32 }
impl Ability for SwimAbility {
    fn describe(&self) -> String { format!("swimming at depth {}", self.depth) }
}

struct GameObject {
    name: String,
    abilities: Vec<Box<dyn Ability>>,
}

fn main() {
    let flying_swimming_enemy = GameObject {
        name: "dragonfly-boss".to_string(),
        abilities: vec![
            Box::new(FlyAbility { altitude: 10.0 }),
            Box::new(SwimAbility { depth: 2.0 }),
        ],
    };

    for ability in &flying_swimming_enemy.abilities {
        println!("{}: {}", flying_swimming_enemy.name, ability.describe());
    }
}
```

วิธีนี้แก้ปัญหา "ต้องเลือกสืบทอดจากไหน" ได้จริง — `GameObject` ไม่ต้องสืบทอดจากอะไรเลย แค่ใส่ `Ability` ที่
ต้องการเข้าไปใน `Vec` (ตรงกับแนวคิด composition ที่ **Part 19** อธิบายไว้พอดี) แต่วิธีนี้มีข้อจำกัดสำคัญที่ทำให้
มันยังไม่เหมาะกับเกมขนาดใหญ่:

1. **หา entity ที่มีความสามารถเฉพาะอย่างไม่มีทางทำได้อย่างมีประสิทธิภาพ** — ถ้าอยากรัน "fly system" กับทุก
   `GameObject` ที่บินได้ในเกมที่มี object เป็นหมื่นตัว คุณต้องวน loop ทุก object แล้วไล่เช็คทีละตัวว่า
   `abilities` มี `FlyAbility` ปนอยู่หรือไม่ (ต้อง downcast จาก `dyn Ability` กลับไปเป็น concrete type ซึ่ง
   ทำได้แต่ช้าและไม่ใช่สิ่งที่ trait object ถูกออกแบบมาให้ทำบ่อย ๆ) ต่างจาก ECS ที่ `Query<&FlyAbility>` กรอง
   entity ที่มี component นี้ให้ทันทีโดยไม่ต้องไล่เช็คเองเลย
2. **ไม่ cache-friendly** — `Box<dyn Ability>` แต่ละตัวกระจายอยู่คนละที่ใน heap (คล้ายปัญหา OOP ที่อธิบายไว้
   ก่อนหน้า) การวน loop จึงกระโดดไปกระโดดมาในหน่วยความจำ ไม่ได้ประโยชน์จาก CPU cache เหมือน ECS ที่เก็บ
   component ชนิดเดียวกันไว้ติดกัน
3. **ความสัมพันธ์ระหว่าง "ความสามารถ" กับ "ข้อมูลอื่น ๆ ของ object" ยังไม่ชัดเจน** — `fly_system` ควรอ่าน/แก้
   `Position` ของ object ด้วยไหม? ใน pattern นี้ต้องส่ง reference ของ `Position` แยกเข้าไปใน method ของ
   `Ability` เอง ทำให้ signature ของ trait บวมขึ้นเรื่อย ๆ ตามจำนวนข้อมูลที่แต่ละความสามารถต้องใช้ร่วมกัน

ECS แก้ปัญหาทั้งสามข้อนี้พร้อมกันด้วยการเปลี่ยนมุมมอง: **component ไม่ใช่ "ความสามารถที่มี method"
แต่เป็น "ข้อมูลเปล่า ๆ ที่ system ภายนอกมาอ่าน"** — เมื่อไม่มี method ผูกอยู่กับข้อมูลเลย การ query "ใครมี
component ชนิดนี้บ้าง" จึงเป็นการค้นหาที่ engine ทำให้อย่างมีประสิทธิภาพได้ในระดับ storage engine ทั้งระบบ
ไม่ใช่การไล่เช็ค type ทีละ object แบบ trait object

#### ECS แก้ปัญหานี้อย่างไร: composition แทน inheritance ในระดับ architecture ทั้งเกม

Entity Component System (ECS) แก้ปัญหานี้ด้วยการพลิกมุมมองทั้งหมด: **ไม่มี class hierarchy เลย** ทุก
"ความสามารถ" หรือ "คุณสมบัติ" ของ game object ถูกแยกเป็น **component** อิสระที่ผูกเข้ากับ entity ได้อย่าง
อิสระ ไม่ว่าจะเป็นชุดไหนก็ตาม:

```
Entity #42 (ศัตรูบินได้และว่ายน้ำได้):
  ├─ Position    { x, y }
  ├─ Health      { current, max }
  ├─ FlyAbility  { altitude }
  ├─ SwimAbility { depth }
  └─ Sprite      { texture }

Entity #43 (ศัตรูเดินธรรมดา):
  ├─ Position { x, y }
  ├─ Health   { current, max }
  └─ Sprite   { texture }

Entity #44 (ผู้เล่น):
  ├─ Position { x, y }
  ├─ Health   { current, max }
  ├─ Player   { } (marker component เปล่า ๆ ไว้บอกว่า "นี่คือผู้เล่น")
  └─ Sprite   { texture }
```

ต้องการศัตรูที่บินได้และว่ายน้ำได้? แค่แนบทั้ง `FlyAbility` และ `SwimAbility` เข้ากับ entity ตัวนั้น — ไม่ต้อง
สร้าง class ใหม่ ไม่ต้อง copy โค้ด ไม่มี diamond problem เพราะไม่มี inheritance ให้ชนกันตั้งแต่แรก แต่ละ
component เป็นอิสระจากกันโดยสมบูรณ์ (ตรงกับสิ่งที่ **Part 9** สอนไว้ว่า struct ไม่มีความสัมพันธ์แบบสืบทอดใน
Rust — struct เป็นแค่ "ที่เก็บข้อมูล" ธรรมดา)

แล้ว "พฤติกรรม" (เช่น การบิน การว่ายน้ำ) อยู่ที่ไหน? นี่คือจุดที่ต่างจาก OOP อย่างชัดเจนที่สุด: ใน OOP
พฤติกรรมอยู่ **ในตัว object** (method ของ class) แต่ใน ECS พฤติกรรมอยู่ **แยกออกมาต่างหาก** เป็น **system**
— ฟังก์ชันธรรมดาที่มองหา entity ทุกตัวที่มี component ชุดที่ต้องการ แล้วทำงานกับมัน:

```
fly_system:     ทำงานกับทุก entity ที่มี (Position, FlyAbility)
swim_system:    ทำงานกับทุก entity ที่มี (Position, SwimAbility)
health_system:  ทำงานกับทุก entity ที่มี (Health)
render_system:  ทำงานกับทุก entity ที่มี (Position, Sprite)
```

Entity #42 ที่มีทั้ง `FlyAbility` และ `SwimAbility` จะถูก `fly_system` และ `swim_system` ประมวลผลทั้งคู่
โดยอัตโนมัติ — ไม่ต้องเขียนเงื่อนไข `if (this instanceof FlyingEnemy)` แบบ OOP เลยแม้แต่บรรทัดเดียว การ
"ประกอบ" ความสามารถเข้ากับ entity เกิดจากการเลือก component ที่จะแนบ ไม่ใช่จากการเลือก class ที่จะสืบทอด — นี่
คือ composition-over-inheritance ในระดับที่ลึกกว่าที่ **Part 19** เคยพูดถึงตอนอธิบายเรื่อง trait เสียอีก เพราะ
มันไม่ใช่แค่ปรัชญาการออกแบบโค้ด แต่เป็น **สถาปัตยกรรมทั้งเอนจิน**

#### ทำไม ECS ยังเร็วกว่าด้วย: มุมมองด้าน performance

นอกจากแก้ปัญหาการออกแบบ ECS ยังมีข้อได้เปรียบด้าน performance ที่สำคัญมาก: เพราะ component ของชนิดเดียวกัน
(เช่น `Position` ของทุก entity) ถูกเก็บติดกันในหน่วยความจำ (แนวคิดที่เรียกว่า "structure of arrays" ตรงข้ามกับ
"array of structs" แบบ OOP ทั่วไป) การให้ system วน iterate `Query<&Position>` จึงเป็นการวน loop อ่าน memory
ที่อยู่ติดกันเป็นแนวยาว ซึ่ง CPU cache ชอบมาก (cache-friendly) ต่างจาก OOP ที่ object กระจายอยู่ทั่ว heap ทำให้
การไล่ตาม pointer แต่ละ object กระโดดไปกระโดดมาในหน่วยความจำ (cache-unfriendly) — นี่คือเหตุผลที่ ECS ถูกใช้ใน
เกมระดับ AAA จำนวนมาก (เช่น Overwatch ของ Blizzard ก็เปิดเผยว่าใช้ ECS ภายใน) ไม่ใช่แค่เพราะดีไซน์สะอาดกว่า
แต่เพราะเร็วกว่าจริงในระดับ hardware

### 103.2 Entity, Component, System: สามเสาหลักของ Bevy ECS

Bevy เป็น game engine ที่เขียนด้วย Rust ทั้งหมด และเลือก ECS เป็นสถาปัตยกรรมหลักตั้งแต่การออกแบบ crate แรกสุด
มาดูสามเสาหลักแบบละเอียด:

#### Entity: แค่ ID เปล่า ๆ ไม่มีข้อมูลอะไรเลย

`Entity` ใน Bevy คือค่าคล้าย ๆ กับ handle หรือ index — ไม่มีข้อมูลอะไรอยู่ในตัวมันเองเลย มันแค่เป็น "กุญแจ" ที่
ใช้เชื่อมโยง component ต่าง ๆ เข้าด้วยกันว่า component ไหนเป็นของ "สิ่งเดียวกัน" ภายในก็คือ index + generation
counter (ป้องกันปัญหา entity เก่าที่ถูกลบไปแล้วแต่ index ถูกเอามาใช้ใหม่กับ entity ใหม่ — คล้ายกับปัญหา
use-after-free ที่ **Part 6-7** สอนเรื่อง ownership ไว้ป้องกัน แต่ Bevy แก้ปัญหานี้ในระดับ data structure ของ
ตัวเองด้วย generation counter แทน)

#### Component: struct ธรรมดาที่ `#[derive(Component)]`

Component คือข้อมูลที่แนบเข้ากับ entity — ในทางปฏิบัติมันก็คือ struct (หรือ enum หรือแม้แต่ unit struct
เปล่า ๆ) ธรรมดาที่ implement trait `Component` ผ่าน derive macro:

```rust
use bevy::prelude::*;

#[derive(Component)]
struct Position {
    x: f32,
    y: f32,
}

#[derive(Component)]
struct Velocity {
    x: f32,
    y: f32,
}

// component ไม่จำเป็นต้องมี field เลยก็ได้ — ใช้เป็น "marker" บอกสถานะ
#[derive(Component)]
struct Player;
```

ข้อสังเกตสำคัญ: component ไม่มีอะไรพิเศษไปกว่า struct ธรรมดาที่เราเรียนมาตั้งแต่ **Part 9** เลย — สิ่งเดียวที่
ทำให้มันเป็น "component" คือการ implement trait `Component` (ผ่าน derive macro ซึ่งเราเรียนกลไกเบื้องหลังไปแล้ว
ใน **Part 45**) เพื่อให้ Bevy's ECS storage รู้จักและเก็บมันได้อย่างมีประสิทธิภาพ

#### System: ฟังก์ชันธรรมดาที่รับ `Query`/`Res`/`Commands` เป็นพารามิเตอร์

System คือฟังก์ชัน Rust ธรรมดา — ไม่ต้อง implement trait อะไรเป็นพิเศษ (Bevy ใช้กลไก generic ที่ซับซ้อนภายใน
เพื่อให้ฟังก์ชันเกือบทุกรูปแบบที่มี signature "ถูกต้อง" กลายเป็น system ได้อัตโนมัติ) system จะรับพารามิเตอร์
พิเศษบางชนิด เช่น `Query<...>`, `Res<...>`, `Commands` และ Bevy จะ "ฉีด" ค่าที่ถูกต้องให้ตอนเรียกระบบนี้ทุกเฟรม
(a pattern ที่เรียกว่า **dependency injection ผ่าน function signature** — Bevy อ่าน type ของแต่ละพารามิเตอร์
ตอน compile time แล้วรู้เองว่าต้องส่งอะไรมาให้)

```rust
use bevy::prelude::*;

# #[derive(Component)]
# struct Position { x: f32, y: f32 }
# #[derive(Component)]
# struct Velocity { x: f32, y: f32 }
#
fn movement_system(mut query: Query<(&mut Position, &Velocity)>) {
    for (mut position, velocity) in &mut query {
        position.x += velocity.x;
        position.y += velocity.y;
    }
}
```

สังเกตว่า `for (mut position, velocity) in &mut query` มีหน้าตาเหมือนการวน loop บน `Vec` ที่เราคุ้นเคยจาก
**Part 25** ทุกประการ — นั่นเพราะ `&mut Query<...>` implement `IntoIterator` เหมือนกับ collection ทั่วไป
เบื้องหลัง Bevy กำลังกรอง entity ทั้งหมดในเกมให้เหลือแค่ตัวที่มีทั้ง `Position` **และ** `Velocity` แล้วคืน
iterator ของ tuple reference (`&mut Position`, `&Velocity`) ของ entity เหล่านั้นออกมา — คุณไม่ต้องเขียน
`if entity.has::<Position>() && entity.has::<Velocity>()` เองเลยแม้แต่บรรทัดเดียว

#### ตัวอย่างสมบูรณ์ตัวแรก: entity เคลื่อนที่ด้วย Position/Velocity

มารวมทุกอย่างเข้าด้วยกันเป็นตัวอย่างที่ compile และรันได้จริง (ตัวอย่างนี้และทุกตัวอย่างในบทนี้ได้ผ่านการ
compile และรันจริงแล้ว รายละเอียดการตรวจสอบอยู่ในหัวข้อ 103.7 และ "กับดักที่พบบ่อย"):

```rust
use bevy::prelude::*;

#[derive(Component)]
struct Position {
    x: f32,
    y: f32,
}

#[derive(Component)]
struct Velocity {
    x: f32,
    y: f32,
}

// system ที่รันครั้งเดียวตอนเริ่มเกม (Startup schedule) — สร้าง entity ตั้งต้น
fn spawn_entities(mut commands: Commands) {
    commands.spawn((
        Position { x: 0.0, y: 0.0 },
        Velocity { x: 1.0, y: 2.0 },
    ));
}

// system ที่รันทุกเฟรม (Update schedule) — อัปเดตตำแหน่งตาม velocity
fn movement_system(mut query: Query<(&mut Position, &Velocity)>) {
    for (mut position, velocity) in &mut query {
        position.x += velocity.x;
        position.y += velocity.y;
    }
}

fn print_position(query: Query<&Position>) {
    for position in &query {
        println!("Position: ({}, {})", position.x, position.y);
    }
}

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Startup, spawn_entities)
        .add_systems(Update, (movement_system, print_position).chain())
        .run();
}
```

อธิบายทีละส่วน:

- `commands.spawn((Position { .. }, Velocity { .. }))` — `Commands` เป็นพารามิเตอร์พิเศษอีกชนิดที่ใช้สั่ง
  "เปลี่ยนแปลงโครงสร้างโลกเกม" เช่นสร้าง/ลบ entity การเปลี่ยนแปลงเหล่านี้ไม่เกิดขึ้นทันที แต่ถูก "จดคิวไว้" แล้ว
  ค่อยประมวลผลจริงตอนจบ schedule ของเฟรมนั้น (เหตุผลเชิง performance: การแก้โครงสร้าง storage กลางเฟรมที่
  system อื่นอาจกำลัง query อยู่พร้อมกันจะทำให้เกิดปัญหาการเข้าถึงข้อมูลที่ไม่ปลอดภัย จึงต้อง defer ไว้ก่อน)
- `commands.spawn((A, B))` — การ spawn entity พร้อม component หลายตัวพร้อมกันทำผ่าน **tuple** ธรรมดา (ตั้งแต่
  Bevy 0.11 เป็นต้นมา ไม่ต้องสร้าง "Bundle" struct แยกอีกต่อไปสำหรับกรณีทั่วไป — tuple ของ component
  implement trait `Bundle` ให้อัตโนมัติผ่าน blanket implementation)
- `App::new().add_plugins(DefaultPlugins)` — สร้าง Bevy application แล้วติดตั้ง "ปลั๊กอิน" มาตรฐานทั้งหมด
  (หน้าต่าง, การ render, เสียง, asset loading, ...) รายละเอียดเต็มอยู่ในหัวข้อ 103.3
- `.add_systems(Startup, spawn_entities)` — ลงทะเบียน system ให้รันใน **Startup schedule** (รันครั้งเดียว
  ตอนเริ่มแอป)
- `.add_systems(Update, (movement_system, print_position).chain())` — ลงทะเบียน 2 system ให้รันใน **Update
  schedule** (รันทุกเฟรม) และ `.chain()` บอกว่าให้รัน `movement_system` ก่อน `print_position` เสมอ (ไม่งั้น
  Bevy อาจเลือกรันสองระบบนี้แบบขนานกันโดยไม่รู้ลำดับ ถ้า `.chain()` ไม่ได้ระบุไว้และไม่มีการเข้าถึงข้อมูลชนกัน
  Bevy จะพยายามรันแบบ multi-thread ให้เร็วที่สุด)
- `.run()` — เริ่ม game loop จริง (บล็อกจนกว่าโปรแกรมจะปิด)

#### Schedule ต่าง ๆ: `Startup`, `Update`, และ `FixedUpdate`

ตัวอย่างข้างบนใช้ไป 2 schedule แล้ว (`Startup` กับ `Update`) แต่ Bevy มี schedule มากกว่านั้น ที่ควรรู้จักเพิ่ม
อีกตัวคือ **`FixedUpdate`** ซึ่งสำคัญมากสำหรับ logic ที่ต้อง "สม่ำเสมอไม่ขึ้นกับความเร็วของเครื่อง" เช่น
physics simulation

ปัญหาที่ `FixedUpdate` แก้คือ: `Update` schedule รันเร็วเท่าที่เครื่องจะรันได้ (ถ้าเครื่องแรงอาจรัน 144
เฟรม/วินาที ถ้าเครื่องช้าอาจรันแค่ 30 เฟรม/วินาที) ถ้าเขียน physics logic ไว้ใน `Update` ตรง ๆ
(`position += velocity` ทุกเฟรมแบบที่ตัวอย่างในบทนี้ทำเพื่อความง่าย) ความเร็วของวัตถุในเกมจะ **ขึ้นกับความเร็ว
ของเครื่องผู้เล่นโดยไม่ตั้งใจ** — เครื่องแรงจะเห็นวัตถุเคลื่อนที่เร็วกว่าเครื่องช้า (เพราะรันระบบถี่กว่า) ซึ่ง
ไม่ fair และทำให้ physics behavior ไม่ deterministic ข้ามเครื่อง

`FixedUpdate` แก้ปัญหานี้โดยรันด้วยความถี่ที่ **กำหนดตายตัว** (ไม่ขึ้นกับเฟรมเรตของการ render) — ถ้าเครื่องแรง
render ได้ 144 fps แต่ตั้ง `FixedUpdate` ไว้ที่ 60Hz ระบบใน `FixedUpdate` ก็จะรันแค่ประมาณ 60 ครั้งต่อวินาที
เท่านั้น (Bevy คำนวณเองว่าต้อง "เร่ง" หรือ "ชะลอ" การเรียก `FixedUpdate` ตามเวลาจริงที่ผ่านไปเทียบกับเวลาที่ควร
จะผ่านไป):

```rust
use bevy::prelude::*;

fn physics_tick(mut count: Local<u32>) {
    *count += 1;
    println!("FixedUpdate tick #{}", *count);
    // ในเกมจริง: อัปเดต velocity ตาม gravity, ตรวจ collision ฯลฯ ที่นี่
}

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .insert_resource(Time::<Fixed>::from_hz(60.0)) // กำหนดความถี่ FixedUpdate เป็น 60 ครั้ง/วินาที
        .add_systems(FixedUpdate, physics_tick)
        .run();
}
```

เราตรวจสอบพฤติกรรมนี้แบบ headless จริง (ใส่ `std::thread::sleep` เล็ก ๆ ระหว่างแต่ละ `app.update()` เพื่อจำลอง
เวลาที่ผ่านไปจริง เนื่องจาก `Time<Fixed>` คำนวณจากเวลานาฬิกาจริงที่ผ่านไป ไม่ใช่จำนวนครั้งที่เรียก
`app.update()`) และยืนยันว่า `physics_tick` ถูกเรียกตามจำนวนเวลาที่ผ่านไปจริงหารด้วยคาบเวลาของ 60Hz ไม่ใช่ตาม
จำนวนครั้งที่ `app.update()` ถูกเรียก — สรุปสั้น ๆ ว่า **`Update` สำหรับ logic ทั่วไปที่ขึ้นกับเฟรมเรตได้ (เช่น
รับ input, อัปเดต UI, สั่ง render), `FixedUpdate` สำหรับ logic ที่ต้อง deterministic ไม่ขึ้นกับเฟรมเรต (เช่น
physics, การจำลองที่ต้องได้ผลเหมือนกันทุกครั้งไม่ว่าเครื่องจะเร็วหรือช้า)**

มินิเกมในหัวข้อ 103.11 ของบทนี้ใช้แค่ `Update` เพื่อความง่าย (เกมง่าย ๆ ระดับนี้ไม่จำเป็นต้องแยก physics ออกมา
เป็น `FixedUpdate`) แต่ถ้าคุณต่อยอดเป็นเกมที่มี physics จริงจังกว่านี้ ควรพิจารณาย้าย logic การเคลื่อนที่ไปไว้
ใน `FixedUpdate` แทน

### 103.3 ติดตั้ง Bevy: `cargo add`, minimal app, เวลา compile และ `dynamic_linking`

#### ติดตั้งด้วย `cargo add bevy`

การเริ่มต้นใช้ Bevy ทำได้ง่ายมากในระดับคำสั่ง:

```bash
cargo new my_bevy_game
cd my_bevy_game
cargo add bevy
```

ณ เวลาที่เขียนบทนี้ (ตรวจสอบจริงกับ crates.io) เวอร์ชันล่าสุดที่เสถียรคือ **Bevy 0.19.1** (มี 0.20.0-rc.1
เป็น release candidate รุ่นถัดไปที่ยังไม่ปล่อยเป็น stable) `cargo add bevy` จะดึงเวอร์ชันนี้มาให้อัตโนมัติ
และเขียนลง `Cargo.toml` เป็น:

```toml
[dependencies]
bevy = "0.19.1"
```

**ข้อควรระวังเรื่อง Rust toolchain**: Bevy 0.19.1 ต้องการ Rust ที่ค่อนข้างใหม่ — ระหว่างตรวจสอบเนื้อหาบทนี้
เราลองรันคำสั่งเดียวกันบนเครื่องที่มี `rustc` เวอร์ชันเก่ากว่าที่ Bevy กำหนด และได้ error message จริงจาก
Cargo แบบนี้:

```
warning: ignoring bevy@0.19.1 (which requires rustc 1.95.0) as it is
incompatible with rustc 1.94.1
      Adding bevy v0.18.1 to dependencies
```

สังเกตว่า Cargo **ไม่ได้ error ทันที** แต่ถอยไปหาเวอร์ชันเก่าลงมา (0.18.1) ที่ยัง compile ได้กับ toolchain
ปัจจุบัน — เป็นพฤติกรรมมาตรฐานของ Cargo dependency resolution (`rust-version` field ใน `Cargo.toml` ของ
dependency ถูกใช้กรองเวอร์ชันที่เลือกได้) ถ้าคุณเจอ warning แบบนี้ วิธีแก้คืออัปเดต Rust toolchain ด้วย
`rustup update stable` ก่อน แล้วค่อย `cargo add bevy` ใหม่เพื่อได้เวอร์ชันล่าสุดจริง ๆ

#### App ขั้นต่ำที่สุด

```rust
use bevy::prelude::*;

fn main() {
    App::new().add_plugins(DefaultPlugins).run();
}
```

โค้ด 3 บรรทัดนี้เพียงพอที่จะเปิดหน้าต่างเปล่า ๆ ขึ้นมาได้แล้ว (บนเครื่องที่มี GPU/display จริง) `DefaultPlugins`
เป็น "plugin group" ที่รวมปลั๊กอินย่อยหลายสิบตัวเข้าไว้ด้วยกัน ครอบคลุมทุกอย่างที่เกมทั่วไปต้องการ: การเปิด
หน้าต่าง (`bevy_winit`), การ render 2D/3D (`bevy_render`, `bevy_core_pipeline`), เสียง (`bevy_audio`), การโหลด
asset (`bevy_asset`), input จากคีย์บอร์ด/เมาส์ (`bevy_input`), เวลา (`bevy_time`) และอื่น ๆ อีกมาก

#### เวลา compile ครั้งแรกของ Bevy: ปัญหาที่โด่งดังในวงการ

Bevy มีชื่อเสียง (ในทางที่ไม่ค่อยดี) เรื่องเวลา compile ครั้งแรกที่ยาวนานมาก เพราะ `DefaultPlugins` ดึง
dependency มาเยอะมาก (การ render แบบ cross-platform ผ่าน `wgpu`, การจัดการ font, การ decode รูปภาพหลาย
format, ...) รวมกันแล้วต้อง compile crate ย่อยหลายร้อยตัว

ระหว่างตรวจสอบเนื้อหาบทนี้ เราวัดเวลา compile จริงในสภาพแวดล้อมตรวจสอบของเราเอง (เครื่องที่ไม่มี GPU/หน้าจอ ใช้
แค่ตรวจ ECS logic แบบ headless — รายละเอียดในหัวข้อ 103.7): การ build ที่ตัด `DefaultPlugins`
(รวมส่วน windowing/rendering) ออก เหลือแค่ core ECS + input + time (`default-features = false, features =
["default_app", "keyboard", "mouse", "multi_threaded"]`) ใช้เวลา **compile ครั้งแรกประมาณ 46 วินาที**
ส่วนเวอร์ชันที่เพิ่ม sprite/camera/transform type เข้ามาด้วย (แต่ยังไม่รวม renderer จริง) ใช้เวลาประมาณ
**1 นาที 41 วินาที** — และนี่เป็นแค่ **เศษเสี้ยว** ของ dependency ทั้งหมดที่ `DefaultPlugins` ใช้จริง (ซึ่งรวม
ทั้ง `wgpu`, `winit`, font rendering แบบเต็มรูปแบบ ที่สภาพแวดล้อมตรวจสอบของเราไม่มี library ระดับระบบปฏิบัติการ
ที่จำเป็นให้ compile ผ่านได้ — รายละเอียดในหัวข้อ 103.7) จากรายงานของทีม Bevy เองและชุมชนผู้ใช้ การ build
`DefaultPlugins` แบบเต็มรูปแบบครั้งแรกบนเครื่อง dev ทั่วไปมักใช้เวลาหลายนาที (ตัวเลขที่รายงานกันทั่วไปอยู่ในช่วง
2-5+ นาที ขึ้นกับสเปคเครื่องและว่าเคย cache dependency ไว้บ้างหรือไม่) — ตัวเลขที่เราวัดได้เองในสภาพแวดล้อม
ตรวจสอบยืนยันว่าแค่ core ECS อย่างเดียว (ไม่มี renderer) ก็ยังกินเวลาหลักสิบวินาทีถึงเป็นนาทีแล้ว ดังนั้นเวอร์ชัน
เต็มที่มี renderer จริงย่อมกินเวลามากกว่านี้แน่นอน

#### บรรเทาด้วย `dynamic_linking` feature (สำหรับ dev เท่านั้น)

Bevy มี feature ชื่อ `dynamic_linking` ที่ช่วยลดเวลา **rebuild ระหว่างพัฒนา** (ไม่ใช่ compile ครั้งแรก) ได้
อย่างมาก เปิดใช้งานด้วย:

```bash
cargo add bevy --features dynamic_linking
```

เราตรวจสอบแล้วว่าชื่อ feature นี้ยังใช้ได้จริงกับ Bevy 0.19.1 (`cargo add` ยอมรับชื่อ feature นี้และ resolve
dependency ได้ปกติ) หลักการทำงานคือ: โดยปกติ Rust จะ **static link** ทุก crate ที่ใช้เข้าไปในไฟล์ binary
เดียว ซึ่งหมายความว่าทุกครั้งที่คุณแก้โค้ดของตัวเองแม้แต่บรรทัดเดียว **linker ต้องเชื่อมทุก crate ของ Bevy
ใหม่ทั้งหมด** เพราะ symbol ของ Bevy ถูกฝังรวมอยู่ใน binary เดียวกับโค้ดคุณ — และ Bevy เป็น crate ขนาดใหญ่มาก
ขั้นตอน linking นี้จึงกินเวลานานผิดปกติเมื่อเทียบกับโปรเจกต์ Rust ทั่วไป

`dynamic_linking` เปลี่ยนให้ Bevy compile เป็น **shared library** (`.so` บน Linux, `.dylib` บน macOS,
`.dll` บน Windows) แยกออกมาต่างหาก แล้วให้ binary ของเกมคุณ **โหลดมันตอน runtime** แทนที่จะฝังเข้าไปตอน
compile — เมื่อคุณแก้โค้ดของตัวเองแล้ว build ใหม่ linker แค่ต้องเชื่อม binary เกมของคุณเข้ากับ shared library
ที่ compile ไว้แล้ว (ไม่ต้อง recompile/relink ตัว Bevy เองซ้ำ) ทำให้ incremental rebuild เร็วขึ้นอย่างมีนัยสำคัญ
(ชุมชน Bevy รายงานว่าลดเวลา rebuild ได้หลายเท่าตัวในหลายกรณี)

**ข้อแลกเปลี่ยนที่ต้องรู้ (เชื่อมโยงกับ Part 96 เรื่อง static vs dynamic linking):** ใน **Part 96** เราพูดถึง
static/dynamic linking ในมุมของ **การ deploy container** — static linking (เช่นด้วย `musl`) ทำให้ binary
สุดท้ายพกพาง่ายไม่ต้องพึ่ง shared library ของระบบปลายทาง แต่ dynamic linking ทำให้ image เล็กลงถ้า base image
มี library ที่ต้องการอยู่แล้ว ในบริบทนั้น การตัดสินใจ static/dynamic คือเรื่อง **production deployment**

แต่ `dynamic_linking` ของ Bevy ในบทนี้เป็นแนวคิดที่ **ตรงข้ามกันโดยสิ้นเชิงในเชิงจุดประสงค์** แม้จะใช้กลไก
เดียวกัน (shared library) — มันคือ **dev-time compile-speed optimization ล้วน ๆ** ไม่เกี่ยวกับการ deploy เลย
สิ่งที่ต้องจำให้ขึ้นใจคือ: **ห้ามใช้ `dynamic_linking` ตอน build สำหรับ production หรือตอนแจกจ่ายเกมให้คนอื่น
เด็ดขาด** เพราะ:

1. binary ที่ได้จะไม่ทำงานถ้าไม่มีไฟล์ `.so`/`.dll` ของ Bevy อยู่ข้าง ๆ (ตรงข้ามกับที่ Part 96 อยากได้ตอน
   deploy คือ binary ที่พึ่งพา library ภายนอกให้น้อยที่สุด)
2. ขนาดและโครงสร้างของ shared library ผูกกับเครื่อง dev ที่ compile มันขึ้นมา ย้ายไปเครื่องอื่นอาจใช้ไม่ได้
3. ทีม Bevy เองก็เตือนชัดเจนในเอกสารว่า feature นี้มีไว้ "สำหรับตอนพัฒนา (development) เท่านั้น"

แนวทางที่ถูกต้องคือใช้ `dynamic_linking` เฉพาะใน dev profile (เช่นผ่าน `[features]` ของ workspace หรือใส่
เป็น feature ที่เปิดแยกด้วยมือตอน `cargo run` ระหว่างพัฒนา) แล้ว **ปิดมันตอน `cargo build --release`** เพื่อ
ให้ binary สุดท้าย static-link ตามปกติ — ตัวอย่างวิธีตั้งค่าใน `Cargo.toml`:

```toml
[dependencies]
bevy = "0.19.1"

# เปิด dynamic_linking เฉพาะตอน dev ด้วยคำสั่ง:
#   cargo run --features bevy/dynamic_linking
# ห้ามใส่ features นี้เข้าไปใน [dependencies] ตรง ๆ แบบถาวร
# เพราะจะติดไปกับ release build ด้วยโดยไม่ตั้งใจ
```

### 103.4 Query เจาะลึก: อ่าน, เขียน, และกรองข้อมูลของ Entity

`Query` คือหัวใจของการเข้าถึงข้อมูลใน ECS — มันคือวิธีที่ system "ขอ" ข้อมูลจากโลกเกมโดยไม่ต้องรู้เลยว่ามี
entity อะไรอยู่บ้าง หรือมีกี่ตัว ระบบแค่บอก "ฉันอยากได้ entity ที่มี component ชุดนี้" แล้ว Bevy จะกรองมาให้เอง

#### `Query<&T>`: อ่านอย่างเดียว

```rust
use bevy::prelude::*;
# #[derive(Component)]
# struct Position { x: f32, y: f32 }

fn print_all_positions(query: Query<&Position>) {
    for position in &query {
        println!("({}, {})", position.x, position.y);
    }
}
```

`Query<&Position>` หมายถึง "ให้ฉัน reference แบบอ่านอย่างเดียวไปยัง `Position` ของทุก entity ที่มี component
นี้" — entity ที่ไม่มี `Position` เลยจะไม่ปรากฏใน query นี้เลย ไม่ต้องกรองเองแม้แต่นิดเดียว

#### `Query<(&mut T, &U)>`: เขียนหนึ่งตัว อ่านอีกตัว ในระบบเดียวกัน

```rust
use bevy::prelude::*;
# #[derive(Component)]
# struct Position { x: f32, y: f32 }
# #[derive(Component)]
# struct Velocity { x: f32, y: f32 }

fn movement_system(mut query: Query<(&mut Position, &Velocity)>) {
    for (mut position, velocity) in &mut query {
        position.x += velocity.x;
        position.y += velocity.y;
    }
}
```

Tuple ใน `Query` (`(&mut Position, &Velocity)`) หมายถึง "เอาแค่ entity ที่มี **ทั้งสอง** component นี้"
(intersection ไม่ใช่ union) — entity ที่มีแค่ `Position` แต่ไม่มี `Velocity` จะไม่ถูกกรองเข้ามาในลูปนี้เลย
สังเกตว่าพารามิเตอร์ตัวแรกของ system ต้องเป็น `mut query: Query<...>` (ตัวแปร `mut`) เพราะเราต้องเรียก
`.iter_mut()`/`for ... in &mut query` ซึ่งต้องยืม mutable reference ออกจากตัว query เอง — ถ้าลืม `mut` ตรงนี้
compiler จะฟ้อง error ทันที (แม้ type ข้างในจะเป็น `&mut Position` ก็ตาม เพราะตัวแปร `query` เองก็ต้อง mutable
ด้วยถ้าจะเรียก mutable iterator ออกมา)

#### `With<T>` และ `Without<T>`: กรองแบบไม่ต้องดึงข้อมูลออกมา

บางครั้งเราต้องการกรอง entity ด้วย component บางตัว แต่ **ไม่ต้องการอ่านค่า** ของมันเลย (เช่นกรองว่า
"ต้องเป็น entity ที่มี marker `Player`" แต่ไม่ต้องใช้ข้อมูลข้างใน `Player` เพราะมันไม่มี field อะไรเลย) — นี่
คือหน้าที่ของ `With<T>`/`Without<T>` ซึ่งใส่เป็น **type parameter ตัวที่สอง** ของ `Query` (ไม่ใช่ปนอยู่ใน
tuple ข้อมูลตัวแรก — จุดนี้เป็นกับดักที่พบบ่อยมาก อ่านรายละเอียดในหัวข้อ "กับดักที่พบบ่อย" ท้ายบท):

```rust
use bevy::prelude::*;
# #[derive(Component)]
# struct Position { x: f32 }
# #[derive(Component)]
# struct Player;
# #[derive(Component)]
# struct Obstacle;

// ได้ Position ของ entity ที่มี Player เท่านั้น (ไม่สนใจ Obstacle เลย)
fn player_only(query: Query<&Position, With<Player>>) {
    for pos in &query {
        println!("player at x={}", pos.x);
    }
}

// ได้ Position ของ entity ที่ "ไม่มี" Obstacle
fn non_obstacles(query: Query<&Position, Without<Obstacle>>) {
    for pos in &query {
        println!("non-obstacle at x={}", pos.x);
    }
}

// รวมหลายเงื่อนไขกรองด้วย tuple ใน type parameter ตัวที่สอง
fn player_not_obstacle(
    query: Query<&Position, (With<Player>, Without<Obstacle>)>,
) {
    for pos in &query {
        println!("{}", pos.x);
    }
}
```

รูปแบบ `Query<Data, Filter>` นี้แยก "ข้อมูลที่จะดึงออกมา" (`Data`, ตัวแรก) กับ "เงื่อนไขกรอง entity"
(`Filter`, ตัวที่สอง) ออกจากกันอย่างชัดเจนในระดับ type system — คอมไพเลอร์ตรวจสอบให้ตั้งแต่ compile time ว่าคุณ
เขียนถูกโครงสร้างหรือไม่ (ลองใส่ `Without<T>` ปนเข้าไปใน `Data` tuple ผิดที่ดูจะเห็น error ทันที เพราะ
`Without<T>` ไม่ implement trait `QueryData` — มันเป็นแค่ filter เท่านั้น)

#### เหตุผลเชิงลึก: ทำไม Bevy ทำแบบนี้ถึงเร็วและปลอดภัย

การที่ system หนึ่งประกาศ `Query<&mut Position, With<Player>>` และอีก system หนึ่งประกาศ
`Query<&mut Position, With<Enemy>>` (สมมติว่า `Player` กับ `Enemy` ไม่มีวันอยู่บน entity เดียวกัน) ทำให้ Bevy
รู้ตั้งแต่ compile time ว่าทั้งสอง query **ไม่มีวันเข้าถึง `Position` ตัวเดียวกัน** ได้ในเวลาเดียวกัน — Bevy
ใช้ข้อมูลนี้ตัดสินใจ **รันสอง system นี้แบบขนานกันบน thread ที่ต่างกันได้อย่างปลอดภัย** โดยอัตโนมัติ (ผ่าน
`multi_threaded` scheduler ที่อาศัยแนวคิด thread pool แบบเดียวกับที่ **Part 37** สอนไว้) นี่คือจุดที่ ECS ของ
Bevy เอาชนะข้อจำกัดของ borrow checker แบบ single-thread ธรรมดา — Rust ทั่วไปจะไม่ยอมให้คุณมี `&mut` สอง
reference ไปยังข้อมูลเดียวกันพร้อมกันแม้จะอยู่ต่าง thread (ตามกฎที่ **Part 7** สอนไว้) แต่ Bevy พิสูจน์ได้ที่
ระดับ query metadata ว่า **ไม่มีทางชนกันจริง** จึงอนุญาตให้รันคู่ขนานได้อย่างปลอดภัยโดยไม่ต้องใช้ `Mutex`
หรือ `RwLock` เลยแม้แต่ตัวเดียว (ต่างจาก `Arc<Mutex<T>>` ที่ **Part 39** สอนไว้ ซึ่งเป็นการป้องกัน "ตอน
runtime" — ในขณะที่ Bevy พยายามพิสูจน์ความปลอดภัยไว้ล่วงหน้า "ตอน schedule build time" แทน)

#### Change Detection: `Added<T>` และ `Changed<T>` — กรอง entity ตามว่า "เพิ่งเปลี่ยน" หรือไม่

Query filter อีกกลุ่มที่มีประโยชน์มากในทางปฏิบัติคือ `Added<T>` และ `Changed<T>` — ทั้งสองไม่ได้กรองตามว่า
entity "มี" component หรือไม่ (แบบ `With`/`Without`) แต่กรองตามว่า component นั้น **เพิ่งถูกเพิ่มเข้ามาในเฟรม
นี้** (`Added<T>`) หรือ **เพิ่งถูกแก้ไขค่าไปในเฟรมนี้** (`Changed<T>`) เทียบกับเฟรมก่อนหน้า

```rust
use bevy::prelude::*;

#[derive(Component)]
struct Position { x: f32 }

#[derive(Component)]
struct Health(i32);

fn setup(mut commands: Commands) {
    commands.spawn((Position { x: 1.0 }, Health(100)));
}

// รันเฉพาะตอนมี entity ที่ "เพิ่งถูก spawn พร้อม Health" ในเฟรมนี้เท่านั้น
fn on_health_added(query: Query<&Health, Added<Health>>) {
    for h in &query {
        println!("new entity with health: {}", h.0);
    }
}

// รันเฉพาะเฟรมที่ค่า Health ถูกแก้ไขจริง (ไม่รันทุกเฟรมเฉย ๆ)
fn on_health_changed(query: Query<&Health, Changed<Health>>) {
    for h in &query {
        println!("health changed to: {}", h.0);
    }
}

fn damage_once(mut query: Query<&mut Health>, mut done: Local<bool>) {
    if *done {
        return; // จำลอง "โดนดาเมจแค่ครั้งเดียว" ด้วย Local<bool> เก็บ state ส่วนตัวของ system นี้
    }
    for mut h in &mut query {
        h.0 -= 10;
    }
    *done = true;
}

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Startup, setup)
        .add_systems(Update, (damage_once, on_health_changed, on_health_added).chain())
        .run();
}
```

เราคอมไพล์และรันโค้ดนี้จริงแบบ headless (สลับ `DefaultPlugins`→`MinimalPlugins` และรัน `app.update()` 3 ครั้ง) ผล
ที่ได้ตรงตามคาด: เฟรมที่ 1 พิมพ์ทั้ง "new entity with health: 100" (จาก `Added`, เพราะ entity ถูก spawn ในเฟรม
นี้เอง) และ "health changed to: 90" (จาก `Changed`, เพราะ `damage_once` แก้ค่าในเฟรมเดียวกัน) — แต่เฟรมที่ 2
และ 3 **ไม่พิมพ์อะไรเลยจากทั้งสอง system** เพราะ `damage_once` มีการเช็ค `Local<bool>` ทำให้ไม่แก้ไขค่าอีกแล้ว
และ `Health` ก็ไม่ได้ "เพิ่งถูกเพิ่ม" อีกต่อไปเช่นกัน

จุดที่ควรสังเกตเพิ่ม: `Local<bool>` ในตัวอย่างนี้เป็นพารามิเตอร์พิเศษอีกชนิดที่ยังไม่ได้พูดถึงมาก่อน — มันคือ
"ตัวแปร state ส่วนตัวที่ผูกกับ system ตัวนั้นตัวเดียว" คงอยู่ข้ามเฟรม แต่ system อื่นมองไม่เห็นและแก้ไม่ได้เลย
(ต่างจาก `Resource` ที่ทุก system เข้าถึงร่วมกันได้) เหมาะกับสถานะภายในเฉพาะของ logic หนึ่งจุด เช่น "ตัวจับเวลา
ภายใน" หรือ "flag ว่าทำสิ่งนี้ไปแล้วหรือยัง" แบบในตัวอย่างนี้

`Added<T>`/`Changed<T>` มีประโยชน์มากในการ optimize — ระบบ UI ที่ต้องอัปเดตข้อความแสดง `Health` บนจอ ไม่
จำเป็นต้องเขียนทับ text ทุกเฟรม (สิ้นเปลืองถ้ามี entity เยอะ) แค่ทำงานเมื่อ `Changed<Health>` เป็นจริงเท่านั้น
ก็พอ — นี่คือรูปแบบการ optimize ที่พบบ่อยมากในเกมจริง

#### เข้าถึง Entity เดี่ยว ๆ: `.get(entity)` และ `.single()`

บางครั้งเราไม่ต้องการวน loop ทุก entity ที่ query เจอ แต่ต้องการ "entity ตัวที่รู้ id อยู่แล้ว" หรือ "entity
ตัวเดียวที่ควรมีอยู่แค่ตัวเดียวในเกม" (เช่นผู้เล่นในเกมคนเดียว) `Query` มี method สำหรับสองกรณีนี้โดยเฉพาะ:

```rust
use bevy::prelude::*;

#[derive(Component)]
struct Position { x: f32 }

#[derive(Component)]
struct Player;

fn setup(mut commands: Commands) {
    let player_entity = commands.spawn((Position { x: 1.0 }, Player)).id();
    println!("spawned entity id: {player_entity:?}");
}

// ใช้ .single() เมื่อ "ควรมี entity แบบนี้อยู่แค่ตัวเดียวในเกม"
fn single_player(query: Query<&Position, With<Player>>) {
    match query.single() {
        Ok(pos) => println!("single player at x={}", pos.x),
        Err(e) => println!("single() error: {e:?}"), // เกิดขึ้นถ้ามี 0 หรือมากกว่า 1 ตัว
    }
}
```

`commands.spawn(...).id()` คืนค่า `Entity` ของ entity ที่เพิ่ง spawn ออกมา — มีประโยชน์เวลาต้องเก็บ id นี้ไว้
ใช้อ้างอิงทีหลัง (เช่นเก็บไว้ใน `Resource` เพื่อจำว่า "นี่คือ entity ของผู้เล่น") เราตรวจสอบแล้วว่า
`Debug`-format ของ `Entity` มีหน้าตาแบบ `13v0` (ตัวเลข index ตามด้วย `v` และเลข generation — ตรงกับที่หัวข้อ
103.2 อธิบายไว้ว่า `Entity` ภายในคือ index + generation counter)

`.single()` คืนค่า `Result<&T, ...>` (ในเวอร์ชันปัจจุบันของ Bevy — เวอร์ชันเก่ากว่านี้บางรุ่นมี `.single()` ที่
panic ตรง ๆ กับมี `.get_single()` แยกไว้คืน `Result` แทน ถ้าเจอโค้ดตัวอย่างเก่าที่ใช้ `get_single()` แล้ว
compile ไม่ผ่าน ให้ลองเปลี่ยนเป็น `.single()` เฉย ๆ ตาม API ปัจจุบัน) — คืน `Err` ถ้า query เจอ entity 0 ตัว
หรือมากกว่า 1 ตัว (เพราะ "single" หมายความว่าต้องมีตัวเดียวเป๊ะ) เหมาะกับสถานการณ์ที่ตามหลักการออกแบบเกมแล้ว
ควรมี entity ชนิดนี้อยู่แค่ตัวเดียวเท่านั้น (กล้องหลัก, ผู้เล่นในเกมคนเดียว) การใช้ `.single()` ทำให้ถ้ามี bug
ที่ spawn entity ซ้ำโดยไม่ตั้งใจ โปรแกรมจะรู้ทันทีผ่านค่า `Err` ที่คืนมา แทนที่จะทำงานผิดเงียบ ๆ

### 103.5 Resource: ข้อมูล global แบบ singleton ที่ไม่ผูกกับ entity ไหนเลย

Component ผูกกับ entity เสมอ — แต่บางข้อมูลในเกมไม่มีเจ้าของเป็น entity ไหนเลย เช่น คะแนนรวมของเกม, เวลาที่
เหลือ, การตั้งค่าความยาก, สถานะ input ของคีย์บอร์ด — ข้อมูลเหล่านี้มีอยู่ **ตัวเดียวสำหรับทั้งเกม** ไม่ใช่แนบ
อยู่กับ entity ตัวใดตัวหนึ่ง นี่คือหน้าที่ของ **Resource**

```rust
use bevy::prelude::*;

#[derive(Resource, Default)]
struct Score(u32);

fn add_score_on_startup(mut score: ResMut<Score>) {
    score.0 += 100;
}

fn print_score(score: Res<Score>) {
    println!("current score: {}", score.0);
}

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .init_resource::<Score>() // สร้าง Score::default() แล้วเก็บไว้เป็น resource
        .add_systems(Startup, add_score_on_startup)
        .add_systems(Update, print_score)
        .run();
}
```

ส่วนประกอบสำคัญ:

- `#[derive(Resource)]` — ทำให้ struct/enum นี้ถูกใช้เป็น resource ได้ (คล้าย `#[derive(Component)]` แต่คนละ
  trait กัน — `Resource` ≠ `Component` และ Bevy เก็บทั้งสองไว้คนละที่กันภายใน)
- `.init_resource::<Score>()` — สร้าง resource ตัวแรกโดยใช้ `Default::default()` (ต้อง derive `Default`
  ด้วยถ้าจะใช้ทางนี้) ถ้าอยากกำหนดค่าเริ่มต้นเองใช้ `.insert_resource(Score(500))` แทนได้
- `Res<Score>` — ยืม resource แบบอ่านอย่างเดียว (คล้าย `&T`)
- `ResMut<Score>` — ยืม resource แบบแก้ไขได้ (คล้าย `&mut T`) — ถ้าใช้ `Res<Score>` แล้วพยายามแก้ไขค่าข้างใน
  compiler จะฟ้อง error ทันที (ดูตัวอย่าง error จริงในหัวข้อ "กับดักที่พบบ่อย")

**ข้อแตกต่างสำคัญระหว่าง Component กับ Resource:**

| | Component | Resource |
|---|---|---|
| จำนวนต่อชนิด | มีได้หลายชุด (หนึ่งชุดต่อ entity) | มีได้แค่ **หนึ่งชุดเดียว** ต่อทั้งแอป |
| เข้าถึงผ่าน | `Query<&T>` (ต้องกรองว่า entity ไหนมีบ้าง) | `Res<T>`/`ResMut<T>` (มีตัวเดียว เข้าถึงตรง ๆ) |
| ตัวอย่างการใช้งาน | ตำแหน่ง, เลือด, sprite ของแต่ละตัว | คะแนนรวม, เวลาเกม, การตั้งค่า, สถานะ input |

ถ้าคุณพยายามสร้าง resource ชนิดเดียวกันซ้ำสองครั้ง (`init_resource::<Score>()` สองรอบ) ครั้งที่สองจะ
**เขียนทับ** ครั้งแรกทันที (resource ไม่ใช่ collection ที่เก็บได้หลายชุดเหมือน component)

### 103.6 Message (ระบบ Event): การสื่อสารแบบ decoupled ระหว่าง System

#### หมายเหตุสำคัญเรื่องชื่อ: Event เดิม vs Message ปัจจุบัน (ตรวจสอบข้าม version จริง)

ก่อนจะสอนวิธีใช้ มีเรื่องสำคัญที่ต้องบอกตรง ๆ ก่อน เพราะเป็นตัวอย่างชัดเจนของสิ่งที่หัวข้อ 103.13 จะพูดถึง:
**API ของ Bevy เปลี่ยนแปลงบ่อยระหว่าง version** เราตรวจสอบ source code จริงของ `bevy_ecs` ข้าม version
เพื่อยืนยันเรื่องนี้ให้แน่ใจ และพบว่า:

- **Bevy 0.16 และเก่ากว่า**: ระบบนี้ชื่อ **`Event`** — ประกาศด้วย `#[derive(Event)]`, อ่านด้วย
  `EventReader<T>`, เขียนด้วย `EventWriter<T>`, ลงทะเบียนด้วย `app.add_event::<T>()`
- **Bevy 0.17 เป็นต้นมา (รวมถึง 0.19.1 ที่บทนี้ใช้)**: ระบบนี้ถูกเปลี่ยนชื่อเป็น **`Message`** — ประกาศด้วย
  `#[derive(Message)]`, อ่านด้วย `MessageReader<T>`, เขียนด้วย `MessageWriter<T>`, ลงทะเบียนด้วย
  `app.add_message::<T>()`

ทำไมถึงเปลี่ยน? เพราะ Bevy เอาชื่อ **`Event`** ไปใช้กับแนวคิดใหม่ที่แยกออกไปต่างหาก คือระบบ **Observer**
(`On<Event>`, ทำงานทันทีตอน trigger ไม่ต้องรอ frame ถัดไปมาอ่าน คล้าย callback มากกว่า queue) ส่วนระบบ "คิว
ข้อความที่ system อื่นมาอ่านทีหลังได้ในเฟรมถัดไป" (ซึ่งเป็นแนวคิดที่บทนี้จะสอน) ถูกเปลี่ยนไปใช้ชื่อ `Message`
แทนเพื่อไม่ให้สับสนกับ Observer ที่ใช้ชื่อ `Event` ไปแล้ว **ถ้าคุณเจอ tutorial หรือโค้ดตัวอย่างเก่าที่ใช้
`EventReader`/`EventWriter`/`add_event` แล้ว compile ไม่ผ่านกับ Bevy เวอร์ชันปัจจุบัน นี่คือสาเหตุ** — ให้
เปลี่ยนเป็น `MessageReader`/`MessageWriter`/`add_message` ตามที่บทนี้สอน

บทนี้จะสอนตาม API ปัจจุบัน (Message) ทั้งหมด เพื่อให้ตรงกับ Bevy 0.19.1 ที่ระบุใน `Cargo.toml`

#### ทำไมต้องมีระบบนี้: decoupling ระหว่าง system

ลองนึกภาพเกมที่เมื่อเกิดการชนกัน (collision) ต้องมีหลายอย่างเกิดขึ้นพร้อมกัน: เพิ่มคะแนน, เล่นเสียงระเบิด,
สร้าง particle effect, อัปเดต achievement — ถ้าเขียนทุกอย่างไว้ใน `collision_check_system` เดียว ระบบนั้นจะ
ใหญ่ขึ้นเรื่อย ๆ และรู้เรื่องของทุกระบบอื่นมากเกินไป (tight coupling)

`Message` แก้ปัญหานี้ด้วยการให้ system ที่ตรวจจับเหตุการณ์ (เช่น การชน) แค่ "ประกาศว่ามันเกิดขึ้นแล้ว" ผ่าน
`MessageWriter` แล้วปล่อยให้ system อื่น ๆ ที่สนใจมาอ่านเอาเองผ่าน `MessageReader` — ระบบที่ตรวจจับไม่จำเป็นต้อง
รู้เลยว่ามีใครฟังอยู่บ้าง กี่ระบบ

**เชื่อมโยงกับ Part 38 (Channels):** นี่คือแนวคิดเดียวกับ `mpsc::channel` ที่ **Part 38** สอนไว้เป๊ะ — "ส่ง
ข้อความแทนการแชร์ state ตรง ๆ" (don't share state, send messages instead) เพียงแต่ต่างบริบทกัน: `mpsc` ใน
Part 38 ใช้ส่งข้อความ **ข้าม OS thread จริง** ที่ทำงานเป็นอิสระจากกัน ในขณะที่ `Message` ของ Bevy ทำงาน
**ภายใน ECS scheduler** — ข้อความถูกเก็บไว้ใน resource ภายใน (`Messages<T>`) ที่ system ทุกตัวใน schedule
เดียวกันเข้าถึงได้ ไม่ต้องส่งผ่าน channel จริงข้าม thread แต่หลักการ "อย่าผูก sender กับ receiver ตรง ๆ ให้ส่ง
ผ่านคิวกลางแทน" เหมือนกันทุกประการ

#### ตัวอย่างจริง: Collision Event ที่สองระบบตอบสนองอย่างอิสระจากกัน

```rust
use bevy::prelude::*;

#[derive(Message)]
struct CollisionEvent {
    #[allow(dead_code)]
    with: Entity,
}

#[derive(Resource, Default)]
struct Score(u32);

// ระบบที่ 1: ได้ยินเรื่องการชน แล้วเพิ่มคะแนน (ไม่รู้เรื่องเสียงเลย)
fn scoring_system(mut reader: MessageReader<CollisionEvent>, mut score: ResMut<Score>) {
    for _ in reader.read() {
        score.0 += 10;
    }
}

// ระบบที่ 2: ได้ยินเรื่องการชนแบบเดียวกัน แล้วเล่นเสียง (ไม่รู้เรื่องคะแนนเลย)
fn sfx_system(mut reader: MessageReader<CollisionEvent>) {
    for _ in reader.read() {
        println!("(sfx) boom!");
    }
}

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .init_resource::<Score>()
        .add_message::<CollisionEvent>() // ต้องลงทะเบียนชนิด message ก่อนใช้งาน!
        .add_systems(Update, (scoring_system, sfx_system))
        .run();
}
```

จุดสำคัญที่ต้องเน้น: **ต้องเรียก `app.add_message::<CollisionEvent>()` ก่อน** ไม่งั้น Bevy จะ panic ตอน
runtime ทันทีที่มี system ใช้ `MessageReader<CollisionEvent>`/`MessageWriter<CollisionEvent>` (รายละเอียด
error message จริงอยู่ในหัวข้อ "กับดักที่พบบ่อย") ทั้ง `scoring_system` และ `sfx_system` ไม่รู้จักกันเลย —
ทั้งคู่แค่ "สมัครฟัง" `CollisionEvent` เหมือนกัน สามารถเพิ่ม/ลบระบบใดระบบหนึ่งได้โดยไม่ต้องแก้อีกระบบเลย นี่คือ
ประโยชน์ของ decoupling ที่ชัดเจนที่สุด

`MessageReader::read()` คืน iterator ของข้อความที่ **ยังไม่เคยอ่านมาก่อน** — Bevy เก็บ "ตัวชี้อ่านล่าสุด" ไว้
ต่อ system ต่อชนิด message (ผ่าน `Local<MessageCursor<T>>` ภายใน) ดังนั้นถ้าสอง system อ่าน message ชนิด
เดียวกัน ทั้งคู่จะเห็นข้อความทุกชิ้นครบ ไม่มีใครแซงคิวไปอ่านก่อนแล้วอีกฝั่ง "พลาด" ข้อความนั้นไป (ต่างจาก
`mpsc::Receiver` ตัวเดียวใน Part 38 ที่ข้อความถูก "กิน" ไปเมื่อมีใครอ่านสำเร็จแล้ว — mental model ของ
`Message` ใกล้เคียงกับ broadcast channel มากกว่า mpsc ตัวเดียว)

#### ข้อความไม่คงอยู่ตลอดไป: double-buffering และเมื่อไหร่ที่ข้อความจะถูกล้าง

รายละเอียดที่ควรรู้อีกจุดคือ: message ที่ไม่มีใครอ่านจะ **ไม่คงอยู่ตลอดไป** — เราตรวจสอบ source code ของ
`bevy_ecs` พบว่า `Messages<T>` ภายในใช้กลยุทธ์ **double buffering** (มี comment ในซอร์สโค้ดยืนยันตรง ๆ ว่า
"Messages are stored in a double buffered queue that switches each frame") หมายความว่าข้อความที่เขียนเข้าไป
ในเฟรมหนึ่ง จะยังอ่านได้ในเฟรมถัดไปด้วย แต่ถ้าไม่มี system ไหนอ่านมันภายใน **2 เฟรม** ข้อความนั้นจะถูกล้างออก
จากคิวไปเลย (ไม่ค้างอยู่ในหน่วยความจำตลอดไปแบบไม่มีที่สิ้นสุด ซึ่งจะเป็นปัญหา memory leak ถ้า Bevy ไม่จัดการ
เรื่องนี้ให้)

ผลเชิงปฏิบัติของเรื่องนี้คือ: **ถ้า system ที่ควรอ่าน message ไม่ได้ถูกเรียกทุกเฟรม** (เช่น system นั้นถูกใส่ไว้
ใน schedule ที่มีเงื่อนไข run condition บางอย่างที่บางเฟรมจะ skip ไป) มีความเป็นไปได้ที่ message จะถูกล้างไป
ก่อนที่ระบบนั้นจะได้อ่านมันเลย นี่ไม่ใช่ปัญหาสำหรับ pattern ปกติที่ system อ่าน message ทุกเฟรมอยู่แล้ว
(แบบตัวอย่างทั้งหมดในบทนี้) แต่เป็นเรื่องที่ควรรู้ไว้ถ้าจะออกแบบระบบที่ซับซ้อนขึ้นในอนาคต

### 103.7 Rendering เบื้องต้น: Camera และ Sprite

#### สปอนเสาร์ Camera2d และ Sprite

ในการ render 2D ด้วย Bevy ต้องมีอย่างน้อยสองอย่างในโลกเกม: กล้อง (camera) และสิ่งที่จะวาด (เช่น sprite)

```rust
use bevy::prelude::*;

#[derive(Component)]
struct Player;

fn setup(mut commands: Commands) {
    // spawn กล้อง 2D — แค่ spawn component เดียว Camera2d ก็พอ
    commands.spawn(Camera2d);

    // spawn entity ที่มี Sprite สีเดียว ขนาด 50x50 พิกเซล ที่ตำแหน่งกึ่งกลางจอ
    commands.spawn((
        Sprite::from_color(Color::srgb(0.2, 0.6, 1.0), Vec2::new(50.0, 50.0)),
        Transform::from_xyz(0.0, 0.0, 0.0),
        Player,
    ));
}

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Startup, setup)
        .run();
}
```

สังเกตว่าเราเขียน `commands.spawn(Camera2d)` โดยไม่ต้องแนบ `Transform`, `Camera`, `Projection` ฯลฯ เข้าไปด้วย
มือเลย ทั้งที่กล้องจริง ๆ ต้องมี component พวกนี้ครบ — เหตุผลคือ Bevy (ตั้งแต่การรื้อระบบ Bundle ใหญ่ในช่วง
0.14-0.15) เปลี่ยนมาใช้กลไกที่เรียกว่า **"required components"**: `Camera2d` ประกาศไว้ในตัวเอง (ผ่าน attribute
`#[require(Camera, Projection::Orthographic(...), Frustum = ...)]` ที่เราตรวจสอบเจอจริงใน source code) ว่า
"ถ้ามีใคร spawn ฉัน ให้แนบ `Camera`, `Projection`, `Frustum` เริ่มต้นให้อัตโนมัติด้วยถ้ายังไม่มี" ผลคือ
`commands.spawn(Camera2d)` เพียงคำสั่งเดียวได้ entity ที่มีทุก component ที่กล้องต้องใช้ครบถ้วน เช่นเดียวกัน
`Sprite` ก็ requires `Transform`, `Visibility` ให้อัตโนมัติ (แต่ในตัวอย่างเราแนบ `Transform::from_xyz(...)`
ไว้เองเพื่อกำหนดตำแหน่งที่ต้องการ ไม่ใช่ปล่อยให้ใช้ค่า default)

`Sprite::from_color(color, size)` เป็น constructor สำเร็จรูปสำหรับสร้าง sprite สี่เหลี่ยมสีล้วน (ไม่ต้องโหลด
ไฟล์ภาพ) เหมาะสำหรับตัวอย่าง/prototype — ถ้าต้องการใช้ภาพจริงจะใช้ `Sprite::from_image(handle)` โดย `handle`
มาจาก `AssetServer` (การโหลด asset เป็นหัวข้อใหญ่ที่นอกเหนือขอบเขตบทนี้ — ดูใน Bevy's official examples ตามที่
หัวข้อสรุปแนะนำ)

#### ข้อจำกัดที่ต้องพูดตรง ๆ: สภาพแวดล้อมตรวจสอบเนื้อหานี้ไม่มีจอแสดงผล

**นี่คือจุดที่ต้องซื่อสัตย์กับผู้อ่านที่สุดในบทนี้:** สภาพแวดล้อมที่เราใช้ตรวจสอบโค้ดทุกตัวอย่างในบทนี้เป็น
sandbox แบบ headless (ไม่มีจอแสดงผล ไม่มี GPU driver, ไม่มี X11/Wayland server) เราตรวจสอบแล้วว่า **ไม่สามารถ
คอมไพล์แม้แต่ backend การเปิดหน้าต่างของ Bevy ได้เต็มรูปแบบ** ในสภาพแวดล้อมนี้ ระหว่างทดสอบเราลอง build
โปรเจกต์ที่ใช้ `DefaultPlugins` เต็มรูปแบบ (ซึ่งเปิดใช้ Wayland backend เป็นค่าเริ่มต้น) และได้ error จริงจาก
build script ของ crate `wayland-sys`:

```
error: failed to run custom build command for `wayland-sys v0.31.11`
...
thread 'main' panicked at .../wayland-sys-0.31.11/build.rs:10:47:
called `Result::unwrap()` on an `Err` value: ...
Package libwayland-client was not found in the pkg-config search path.
```

สาเหตุคือ sandbox นี้ไม่มี development header ของ Wayland ติดตั้งไว้ (แม้จะมี runtime `.so` library อยู่บ้าง
แต่ไม่มีไฟล์ `.pc` ของ pkg-config ที่ build script ต้องใช้ค้นหา) เราลองปิด feature `wayland` แล้วเปิดเฉพาะ
`x11` แทนดู พบว่า build ผ่านขั้นตอน `wayland-sys` ไปได้ไกลกว่าเดิมมาก (compile ผ่าน `bevy_render`,
`bevy_winit` ไปจนถึงขั้น link) แต่สุดท้ายก็ชนข้อจำกัดพื้นที่ดิสก์ของ sandbox ที่ใช้ร่วมกันระหว่าง agent หลายตัว
(`error: failed to build archive ...: No space left on device`) ก่อนจะ build เสร็จสมบูรณ์

สิ่งที่เราตรวจสอบได้จริงและยืนยันได้ 100% คือ:

1. **โค้ดที่ spawn `Camera2d`/`Sprite`/`Transform` ข้างบน type-check ผ่านและรันได้จริง** — เราคอมไพล์ด้วย
   feature set ที่ดึงเฉพาะ **ชนิดข้อมูล** ของ sprite/camera/transform เข้ามา (feature `2d_api` ของ crate
   `bevy` ซึ่งแยก "ชนิดข้อมูล" ออกจาก "ตัว renderer จริงที่ใช้ GPU" — Bevy จัดโครงสร้าง feature ไว้ละเอียด
   ระดับนี้) โดยไม่ดึง `wgpu`/`winit` เข้ามาเลย แล้ว spawn entity ตามโค้ดข้างบนจริงในแบบ headless (ใช้
   `MinimalPlugins` แทน `DefaultPlugins`) จากนั้น query กลับมาตรวจว่า entity มี component ที่ถูกต้องครบ
   (`Sprite`, `Transform`, `Player`) — ผลคือ query เจอ entity ตรงตามที่คาดไว้ทุกประการ
2. **สิ่งที่เราตรวจสอบไม่ได้เลยคือ "หน้าตาของภาพที่ render จริงบนจอ"** — เพราะไม่มี GPU/display ในสภาพแวดล้อม
   นี้ นี่คือปัญหาที่ต่างจากบทเรื่อง WASM/web ก่อนหน้า (**Part 86-95**) อย่างสิ้นเชิง: บทเรื่อง web ยังพอใช้
   headless Chromium render หน้าเว็บแล้ว capture ผลลัพธ์ที่เป็น HTML/DOM ได้ เพราะเว็บเบราว์เซอร์ยังมี software
   rendering path ที่ไม่ง้อ GPU จริงในหลายกรณี แต่ Bevy เป็นแอป native ที่ render ผ่าน GPU โดยตรงผ่าน `wgpu`
   (Vulkan/Metal/DirectX/WebGPU ขึ้นกับ platform) ซึ่งเป็นปัญหาการตรวจสอบคนละชั้นกันโดยสิ้นเชิง — ไม่มี "GPU
   เสมือนที่รันโดยไม่มี GPU จริง" ให้ใช้แบบง่าย ๆ เหมือน headless browser

ดังนั้นถ้าคุณรันโค้ดตัวอย่างนี้บนเครื่องของคุณเองที่มีจอแสดงผลจริง สิ่งที่คุณควรเห็นคือ: หน้าต่างเปล่า ๆ
เปิดขึ้นมา พื้นหลังสีเทาเข้ม (ค่า default ของ Bevy) และมีสี่เหลี่ยมสีฟ้าขนาด 50x50 พิกเซลอยู่กึ่งกลางจอ — แต่
**นี่คือคำอธิบายจากความรู้เรื่อง Bevy ทั่วไป ไม่ใช่ผลที่เราจับภาพหรือเห็นจริงในสภาพแวดล้อมตรวจสอบนี้** เราจะ
ไม่เสกภาพหรือ pixel ที่ไม่ได้เห็นจริงมาบรรยายให้ฟังดูเหมือนได้ตรวจสอบแล้ว — สิ่งที่ตรวจสอบแล้วจริงคือ "โค้ด
compile และ ECS logic (entity/component ที่ถูก spawn) ทำงานถูกต้อง" เท่านั้น

### 103.8 Input Handling: `ButtonInput<KeyCode>`

Bevy จัดการ input ผ่าน resource ชื่อ `ButtonInput<T>` (พารามิเตอร์ generic `T` เปลี่ยนไปตามชนิด input เช่น
`KeyCode` สำหรับคีย์บอร์ด, `MouseButton` สำหรับเมาส์) — **ข้อควรระวังเรื่องชื่ออีกครั้ง**: ใน Bevy รุ่นเก่ามาก
(ก่อน 0.14) resource นี้เคยชื่อ `Input<T>` เฉย ๆ แต่ถูกเปลี่ยนชื่อเป็น `ButtonInput<T>` มานานแล้วเพื่อความชัดเจน
ขึ้น (สื่อว่าเป็น input แบบ "ปุ่มที่กด/ไม่กด" ไม่ใช่ input ทั่วไปทุกชนิด) — บทนี้สอนตามชื่อปัจจุบัน
`ButtonInput<T>` ซึ่งตรวจสอบแล้วว่าตรงกับ Bevy 0.19.1 (`grep` หา struct จริงใน `bevy_input` crate เจอ
`pub struct ButtonInput<T>` ยืนยันชัดเจน)

```rust
use bevy::prelude::*;

#[derive(Component)]
struct Player;

fn player_movement(
    input: Res<ButtonInput<KeyCode>>,
    mut query: Query<&mut Transform, With<Player>>,
) {
    let mut direction = Vec2::ZERO;

    if input.pressed(KeyCode::ArrowLeft) {
        direction.x -= 1.0;
    }
    if input.pressed(KeyCode::ArrowRight) {
        direction.x += 1.0;
    }
    if input.just_pressed(KeyCode::Space) {
        println!("jump!");
    }

    for mut transform in &mut query {
        transform.translation.x += direction.x * 5.0;
    }
}

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Update, player_movement)
        .run();
}
```

Method สำคัญของ `ButtonInput<T>`:

- `.pressed(key)` — คืน `true` ตลอดเวลาที่ปุ่มนั้นถูกกดอยู่ (ใช้กับการเดินต่อเนื่อง)
- `.just_pressed(key)` — คืน `true` เฉพาะ **เฟรมแรก** ที่ปุ่มถูกกด (ใช้กับ action ที่ทำครั้งเดียวต่อการกด
  เช่นกระโดด, ยิง)
- `.just_released(key)` — คืน `true` เฉพาะเฟรมแรกที่ปล่อยปุ่ม

#### ตรวจสอบ input แบบ headless: จำลองการกดปุ่มด้วยมือ

ข่าวดีคือส่วนนี้เราตรวจสอบได้แบบ headless เต็มรูปแบบ เพราะ `ButtonInput<T>` เป็นแค่ resource ธรรมดาที่เก็บ
`HashSet` ของปุ่มที่กดอยู่ — ไม่ต้องพึ่ง OS event loop จริงเลยถ้าเราจำลองการกดปุ่มด้วยมือผ่าน method
`.press()`/`.release()` ตรง ๆ (นี่คือวิธีเดียวกับที่ทีม Bevy เองใช้เขียน integration test ภายในของตัวเอง):

```rust
use bevy::prelude::*;

fn main() {
    let mut app = App::new();
    app.add_plugins(MinimalPlugins);

    // จำลอง resource ของ input เองแบบ manual (ปกติ DefaultPlugins จะสร้างให้)
    app.world_mut()
        .insert_resource(ButtonInput::<KeyCode>::default());

    // จำลองว่าผู้เล่นกดปุ่มลูกศรขวาไว้
    app.world_mut()
        .resource_mut::<ButtonInput<KeyCode>>()
        .press(KeyCode::ArrowRight);

    app.add_systems(Update, |input: Res<ButtonInput<KeyCode>>| {
        if input.pressed(KeyCode::ArrowRight) {
            println!("moving right! (simulated key press detected)");
        }
    });

    app.update(); // รัน 1 เฟรม — ระบบข้างบนจะเห็นว่าปุ่มถูกกดอยู่จริง
}
```

เราคอมไพล์และรันโค้ดนี้จริงแล้ว (ผลลัพธ์: `moving right! (simulated key press detected)` พิมพ์ออกมาตามคาด)
ยืนยันว่า logic การตรวจจับ input ทำงานถูกต้อง — สิ่งที่ตรวจสอบแบบนี้ไม่ครอบคลุมคือการแปลง OS keyboard event
จริงเป็นการเรียก `.press()` (ซึ่งเป็นงานของ `bevy_winit` ที่ผูกกับหน้าต่างจริง ไม่ใช่ ECS logic)

### 103.9 จัดโครงสร้างโค้ดด้วย `Plugin` trait

ตัวอย่างทั้งหมดที่ผ่านมาลงทะเบียน system ตรงใน `main()` ผ่าน `.add_systems(...)` ต่อ ๆ กันไป ซึ่งใช้ได้ดีตอน
โปรเจกต์ยังเล็ก แต่พอเกมโตขึ้นเรื่อย ๆ `main()` จะยาวและอ่านยากขึ้นตามไปด้วย Bevy มีวิธีจัดกลุ่ม system/resource/
message ที่เกี่ยวข้องกันไว้ด้วยกันผ่าน trait `Plugin` — เทียบได้กับการแยกโค้ดเป็น module ตามที่ **Part 16**
สอนไว้ แต่ในระดับที่ผูกกับ "การลงทะเบียนเข้ากับ `App`" โดยตรง

```rust
use bevy::prelude::*;

#[derive(Component)]
struct Position { x: f32, y: f32 }

#[derive(Component)]
struct Player;

// รวม state/system ที่เกี่ยวกับ "การเคลื่อนที่" ไว้ด้วยกันเป็น plugin เดียว
struct MovementPlugin;

impl Plugin for MovementPlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(Startup, setup)
            .add_systems(Update, move_right);
    }
}

fn setup(mut commands: Commands) {
    commands.spawn((Position { x: 0.0, y: 0.0 }, Player));
}

fn move_right(mut query: Query<&mut Position>) {
    for mut pos in &mut query {
        pos.x += 1.0;
    }
}

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_plugins(MovementPlugin) // ลงทะเบียนทั้ง setup + move_right ในคำสั่งเดียว
        .run();
}
```

`Plugin` เป็น trait ที่มี method บังคับตัวเดียวคือ `build(&self, app: &mut App)` — เขียนอะไรก็ได้ข้างในที่ทำกับ
`app` ได้ตามปกติ (เพิ่ม system, resource, message, plugin ย่อยอื่น ๆ ซ้อนกันไปอีกก็ได้) แล้วเรียกใช้ผ่าน
`.add_plugins(MovementPlugin)` เพียงคำสั่งเดียว — `DefaultPlugins` ที่เราใช้มาตลอดทั้งบทก็คือตัวอย่างของ
"plugin group" ขนาดใหญ่ที่รวม plugin เล็ก ๆ หลายสิบตัวไว้ด้วยกันแบบเดียวกันนี้เป๊ะ

ประโยชน์เชิงโครงสร้างคือ: ทีมพัฒนาเกมขนาดใหญ่มักแยกแต่ละ "ระบบย่อยของเกม" (เช่น `PlayerPlugin`,
`EnemyPlugin`, `UiPlugin`, `AudioPlugin`) ไว้เป็น plugin ของตัวเอง คนละไฟล์ คนละ module ตาม **Part 16-17**
แล้วประกอบเข้าด้วยกันใน `main()` เป็นรายการ `.add_plugins((PlayerPlugin, EnemyPlugin, UiPlugin))` สั้น ๆ
อ่านเข้าใจง่ายว่าเกมประกอบด้วยระบบอะไรบ้างในภาพรวม โดยไม่ต้องไล่เปิดดู system ทุกตัวทีละบรรทัด

### 103.10 State Management: จัดการสถานะเกมด้วย `States`

เกมส่วนใหญ่มีสถานะระดับสูงที่ต้องสลับไปมา เช่น "อยู่หน้าเมนู" → "กำลังเล่น" → "หยุดชั่วคราว" → "จบเกม" — ใน
หัวข้อ 103.11 ถัดไป มินิเกมของเราจะใช้แนวคิดนี้แบบง่ายที่สุดผ่าน `enum GameState { Playing, GameOver }` ที่
เก็บเป็น `Resource` ธรรมดาแล้วเช็คด้วย `if` ทุกเฟรม แต่ Bevy มีระบบที่ออกแบบมาเฉพาะสำหรับเรื่องนี้ชื่อ
**`States`** ซึ่งให้ประโยชน์เพิ่มเติมคือระบบ schedule พิเศษที่รันเฉพาะ "ตอนเปลี่ยนสถานะ"
(`OnEnter`/`OnExit`) แทนที่ต้องเช็คด้วยมือทุกเฟรม

```rust
use bevy::prelude::*;

// #[derive(States)] ต้องมี Debug, Clone, PartialEq, Eq, Hash, Default ครบ
// (ผูกกับ Part 10 เรื่อง enum + Part 15 เรื่อง Hash สำหรับใช้เป็น key)
#[derive(States, Debug, Clone, Copy, Eq, PartialEq, Hash, Default)]
enum AppState {
    #[default]
    Menu,
    InGame,
}

fn on_enter_game() {
    println!("entered InGame state — spawn player, obstacles, reset score ที่นี่");
}

fn on_exit_game() {
    println!("left InGame state — cleanup entities ที่นี่");
}

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .init_state::<AppState>() // ต้องเรียกก่อนใช้ OnEnter/OnExit/NextState
        .add_systems(OnEnter(AppState::InGame), on_enter_game)
        .add_systems(OnExit(AppState::InGame), on_exit_game)
        .run();
}
```

การสั่งเปลี่ยนสถานะทำผ่าน resource พิเศษ `NextState<AppState>` (ไม่ใช่การแก้ resource ของ enum ตรง ๆ) —
ตัวอย่างระบบที่กดปุ่ม Enter แล้วเปลี่ยนจากเมนูไปเข้าเกม:

```rust
use bevy::prelude::*;
# #[derive(States, Debug, Clone, Copy, Eq, PartialEq, Hash, Default)]
# enum AppState { #[default] Menu, InGame }

fn menu_input(
    input: Res<ButtonInput<KeyCode>>,
    mut next_state: ResMut<NextState<AppState>>,
) {
    if input.just_pressed(KeyCode::Enter) {
        next_state.set(AppState::InGame);
    }
}
```

เราตรวจสอบโค้ดในหัวข้อนี้แบบ headless จริง (ต้องเพิ่ม `bevy::state::app::StatesPlugin` เข้าไปคู่กับ
`MinimalPlugins` เพราะ `DefaultPlugins` เท่านั้นที่รวม state plugin ให้อัตโนมัติ, `MinimalPlugins` ไม่รวม) โดย
เรียก `app.world_mut().resource_mut::<NextState<AppState>>().set(AppState::InGame)` แทนการจำลอง key press แล้ว
เรียก `app.update()` — ผลลัพธ์จริงที่ได้คือข้อความ `"entered InGame state"` ถูกพิมพ์ออกมาแค่ **ครั้งเดียว** ใน
เฟรมที่สถานะเปลี่ยนจริง (ไม่ใช่ทุกเฟรมที่ `AppState::InGame` เป็นสถานะปัจจุบัน) ยืนยันว่า `OnEnter` ทำงานเป็น
"trigger ตอนเปลี่ยนสถานะ" จริงตามที่ออกแบบไว้ ไม่ใช่ระบบ polling ที่เช็คทุกเฟรมแบบ `if state == X` ที่มินิเกม
ในหัวข้อ 103.11 ใช้

**เมื่อไหร่ควรใช้ `States` เต็มรูปแบบ เทียบกับ `enum` เก็บใน `Resource` ธรรมดา:** สำหรับเกมเล็ก ๆ ที่มีสถานะ
ไม่กี่แบบและเช็คแค่ 1-2 จุด (แบบมินิเกมในบทนี้) การเก็บ enum เป็น resource ธรรมดาก็เพียงพอและเข้าใจง่ายกว่า —
แต่พอเกมมีสถานะซับซ้อนหลายระดับ (เมนูหลัก → เมนูเลือกด่าน → กำลังเล่น → หยุดชั่วคราว → จบเกม → หน้าคะแนน) และ
ต้องมี logic "ทำครั้งเดียวตอนเข้า/ออกจากสถานะ" กระจายอยู่หลายจุด ระบบ `States` เต็มรูปแบบจะช่วยลดโค้ดซ้ำซ้อน
(`if state == X && !already_did_this`) ได้มาก

### 103.11 มินิเกมสมบูรณ์: Dodge the Falling Obstacle

มารวมทุกอย่างที่เรียนมาเข้าด้วยกัน: entity, component, system, resource, message, และ input ในเกมเล็ก ๆ ที่
สมบูรณ์ — กติกา: ผู้เล่นควบคุมตำแหน่งแนวนอนด้วยลูกศรซ้าย/ขวา มีสิ่งกีดขวางตกลงมาจากด้านบน ถ้าสิ่งกีดขวางตกถึง
พื้น (`y <= 0`) ในตำแหน่ง x เดียวกับผู้เล่น เกมจะจบและหยุดเพิ่มคะแนน

```rust
use bevy::prelude::*;

#[derive(Component)]
struct Position {
    x: i32,
    y: i32,
}

#[derive(Component)]
struct Velocity {
    y: i32,
}

#[derive(Component)]
struct Player;

#[derive(Component)]
struct Obstacle;

#[derive(Resource, Default)]
struct Score(u32);

#[derive(Resource, Default, PartialEq, Debug)]
enum GameState {
    #[default]
    Playing,
    GameOver,
}

#[derive(Message)]
struct CollisionEvent;

// system 1: อ่าน input แล้วขยับผู้เล่นในแนวนอน
fn player_input_system(
    input: Res<ButtonInput<KeyCode>>,
    mut query: Query<&mut Position, With<Player>>,
) {
    for mut pos in &mut query {
        if input.pressed(KeyCode::ArrowLeft) {
            pos.x -= 1;
        }
        if input.pressed(KeyCode::ArrowRight) {
            pos.x += 1;
        }
    }
}

// system 2: สิ่งกีดขวางตกลงมาตาม velocity ของมัน
fn obstacle_fall_system(mut query: Query<(&mut Position, &Velocity), With<Obstacle>>) {
    for (mut pos, vel) in &mut query {
        pos.y += vel.y;
    }
}

// system 3: ตรวจการชน (เฉพาะตอนเกมยังเล่นอยู่) แล้วส่ง CollisionEvent ออกไป
// สังเกตการใช้ With/Without คู่กันเพื่อทำให้สอง Query นี้ "แยกกันเด็ดขาด"
// (player_q ไม่มีวันเจอ entity ตัวเดียวกับ obstacle_q) - Bevy จึงอนุญาตให้มีสอง
// Query แบบนี้อยู่ในระบบเดียวกันได้โดยไม่ error แม้จะดู "ทับกัน" ที่ชนิดข้อมูล Position
fn collision_check_system(
    state: Res<GameState>,
    player_q: Query<&Position, (With<Player>, Without<Obstacle>)>,
    obstacle_q: Query<&Position, (With<Obstacle>, Without<Player>)>,
    mut writer: MessageWriter<CollisionEvent>,
) {
    if *state != GameState::Playing {
        return;
    }
    for player_pos in &player_q {
        for obstacle_pos in &obstacle_q {
            if player_pos.x == obstacle_pos.x && obstacle_pos.y <= 0 {
                writer.write(CollisionEvent);
            }
        }
    }
}

// system 4: ถ้ามี CollisionEvent เข้ามา ให้เปลี่ยนสถานะเป็น GameOver
fn game_over_system(mut reader: MessageReader<CollisionEvent>, mut state: ResMut<GameState>) {
    for _ in reader.read() {
        *state = GameState::GameOver;
        println!("GAME OVER: player was hit by an obstacle!");
    }
}

// system 5: เพิ่มคะแนนทุกเฟรมที่ยังเล่นอยู่
fn scoring_system(state: Res<GameState>, mut score: ResMut<Score>) {
    if *state == GameState::Playing {
        score.0 += 1;
    }
}

fn setup(mut commands: Commands) {
    commands.spawn((Position { x: 0, y: 0 }, Player));
    commands.spawn((Position { x: 0, y: 5 }, Velocity { y: -1 }, Obstacle));
}

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .init_resource::<Score>()
        .init_resource::<GameState>()
        .add_message::<CollisionEvent>()
        .add_systems(Startup, setup)
        .add_systems(
            Update,
            (
                player_input_system,
                obstacle_fall_system,
                collision_check_system,
                game_over_system,
                scoring_system,
            )
                .chain(), // ต้องรันตามลำดับนี้เป๊ะ ๆ: input -> fall -> check -> game_over -> score
        )
        .run();
}
```

จุดออกแบบที่ควรสังเกต:

- `.chain()` สำคัญมากในตัวอย่างนี้ — ถ้าไม่ chain ไว้ Bevy อาจรัน `scoring_system` ก่อน `game_over_system`
  ในบางเฟรม ทำให้คะแนนเพิ่มไปอีกหนึ่งครั้งในเฟรมที่เกมจบไปแล้ว (race condition เชิง logic แม้จะไม่มี data
  race ในเชิง memory เลยก็ตาม — Bevy ป้องกัน data race ให้แล้วโดยธรรมชาติของ borrow checker แต่ **ลำดับ
  ทาง logic** ยังเป็นเรื่องที่ผู้เขียนเกมต้องคุมเอง)
- `player_q`/`obstacle_q` ใน `collision_check_system` ดูเหมือนจะ "ชนกัน" เพราะทั้งคู่ query `&Position` แต่
  จริง ๆ ไม่ชนกันเลยเพราะ filter `With<Player>, Without<Obstacle>` กับ `With<Obstacle>, Without<Player>`
  การันตีทาง type system ว่า entity ที่เข้า query แรกไม่มีวันเข้า query ที่สองได้ — Bevy ยอมรับ pattern นี้
  โดยไม่ panic (ต่างจากถ้าเราลบ `Without` ทั้งสองออกแล้วปล่อยให้ query อาจจะทับกันได้ — ดูตัวอย่าง error จริง
  ของกรณีนี้ในหัวข้อกับดัก)

#### ผลการตรวจสอบ headless จริง (ไม่ใช่การจำลอง)

เราคอมไพล์และรันโค้ดเกมนี้จริง (สลับ `DefaultPlugins` เป็น `MinimalPlugins` เพื่อรันแบบ headless ไม่มีหน้าต่าง
พร้อมเรียก `app.update()` เอง 6 ครั้งแทนการ `.run()` วนลูปตลอดไป) ผลลัพธ์จริงที่พิมพ์ออกมา:

```
frame 1: state=Playing score=1
frame 2: state=Playing score=2
frame 3: state=Playing score=3
frame 4: state=Playing score=4
GAME OVER: player was hit by an obstacle!
frame 5: state=GameOver score=4
frame 6: state=GameOver score=4
MINI-GAME HEADLESS VERIFICATION PASSED: final score = 4
```

อธิบายผลลัพธ์: obstacle เริ่มที่ `y=5` ตกลงมาทีละ 1 หน่วยต่อเฟรม (`velocity.y = -1`) ผู้เล่นไม่ได้กดปุ่มอะไร
เลยจึงอยู่ที่ `x=0` ตลอด — เมื่อ obstacle ตกถึง `y=1` (หลังเฟรมที่ 4) ยังไม่ชน (`y <= 0` ยังไม่จริง) แต่หลัง
เฟรมที่ 5 obstacle อยู่ที่ `y=0` ซึ่งตรงกับเงื่อนไข `y <= 0` และ `x` ตรงกับผู้เล่น (`0 == 0`) พอดี ทำให้เกิด
collision และเปลี่ยนสถานะเป็น `GameOver` ที่เฟรมที่ 5 — สังเกตว่าคะแนนหยุดเพิ่มที่ 4 ตั้งแต่เฟรมที่ 5 เป็นต้นไป
(เพราะ `scoring_system` เช็ค `GameState::Playing` ก่อนเพิ่มคะแนนทุกครั้ง) เราได้ `assert_eq!` ตรวจสอบค่า
`GameState::GameOver` และ `score.0 == 4` ไว้ในโค้ดตรวจสอบจริง และมัน **ผ่านทั้งคู่** — นี่คือหลักฐานที่ยืนยัน
ได้ว่า ECS logic ทั้งหมด (input, movement, collision detection, message-driven state transition, resource
update) ทำงานถูกต้องตามที่ออกแบบไว้ทุกประการ โดยไม่ต้องมีจอแสดงผลเลยแม้แต่นิดเดียว

ถ้าคุณรันเกมนี้จริงบนเครื่องที่มีจอแสดงผล (สลับกลับไปใช้ `DefaultPlugins` และเพิ่ม `Camera2d`/`Sprite` ตาม
หัวข้อ 103.7 เพื่อให้เห็นภาพจริง) สิ่งที่ควรเกิดขึ้นคือ: เห็นสี่เหลี่ยมตกลงมาจากด้านบนของจอ ผู้เล่นขยับซ้าย/ขวา
ได้ด้วยลูกศร และเมื่อสี่เหลี่ยมตกถึงพื้นตรงตำแหน่งผู้เล่น คอนโซลจะพิมพ์ "GAME OVER" — ตรรกะเบื้องหลังทั้งหมดที่
ทำให้สิ่งนี้เกิดขึ้นได้ **ตรวจสอบแล้วว่าทำงานถูกต้องจริง** (ตามผลลัพธ์ headless ข้างบน) มีแค่ "ภาพที่ปรากฏบนจอ"
เท่านั้นที่เราไม่ได้เห็นด้วยตาตัวเองในสภาพแวดล้อมตรวจสอบนี้

### 103.12 กลไก Headless Testing ของ Bevy: `MinimalPlugins` และ `app.update()`

ทำไมโค้ดในหัวข้อ 103.11 ถึงรันแบบ headless ได้? กลไกสำคัญที่ Bevy มีมาให้ (และทีม Bevy เองก็ใช้เขียน
integration test ภายในโปรเจกต์ตัวเอง) มีสองส่วนหลัก:

#### `MinimalPlugins`: plugin group ที่มีแค่สิ่งที่ ECS ต้องการ

`MinimalPlugins` เป็น plugin group ที่ตรงข้ามกับ `DefaultPlugins` โดยสิ้นเชิง — เราตรวจสอบ source code จริง
พบว่ามันประกอบด้วยแค่ 4 ปลั๊กอิน:

```
MinimalPlugins:
  - TaskPoolPlugin   (ตั้งค่า thread pool สำหรับ multi_threaded scheduler)
  - FrameCountPlugin (นับจำนวนเฟรมที่ผ่านไป)
  - TimePlugin       (จัดการเวลาระหว่างเฟรม)
  - ScheduleRunnerPlugin (ตัวควบคุมการวน game loop — ปกติ DefaultPlugins ให้ winit event loop
                          เป็นคนควบคุมแทน แต่ MinimalPlugins ต้องมีตัวนี้เอง)
```

ไม่มี windowing, ไม่มี rendering, ไม่มีเสียง, ไม่มี asset loading เลย — เหมาะสำหรับ:

1. เขียน automated test สำหรับ ECS logic (แบบที่บทนี้ทำ)
2. รันเกมบน server (เช่น dedicated game server ที่ไม่ต้องแสดงผลอะไรเลย รอแค่ประมวลผล state แล้วส่งผ่าน
   network)
3. รันใน CI pipeline ที่ไม่มี GPU

#### ควบคุมเฟรมด้วยมือผ่าน `app.update()`

โดยปกติ `App::run()` จะบล็อกโปรแกรมและวน loop ไปเรื่อย ๆ (ควบคุมโดย `ScheduleRunnerPlugin` หรือ event loop
ของ `winit`) แต่สำหรับการทดสอบ เราต้องการควบคุมว่า "รันไปกี่เฟรมแล้วตรวจผลลัพธ์" — `App` มี method `update()`
สาธารณะ (`pub fn update(&mut self)`) ที่รันแค่ **หนึ่งเฟรมเดียว** แล้วคืนการควบคุมกลับมาทันที ทำให้เราเขียน
โค้ดแบบนี้ได้:

```rust
use bevy::prelude::*;

# #[derive(Component)]
# struct Position { x: f32, y: f32 }
# #[derive(Component)]
# struct Velocity { x: f32, y: f32 }
# fn spawn_entities(mut commands: Commands) {
#     commands.spawn((Position { x: 0.0, y: 0.0 }, Velocity { x: 1.0, y: 2.0 }));
# }
# fn movement_system(mut query: Query<(&mut Position, &Velocity)>) {
#     for (mut position, velocity) in &mut query {
#         position.x += velocity.x;
#         position.y += velocity.y;
#     }
# }
#
fn main() {
    let mut app = App::new();
    app.add_plugins(MinimalPlugins)
        .add_systems(Startup, spawn_entities)
        .add_systems(Update, movement_system);

    for _ in 0..3 {
        app.update(); // รันทีละเฟรม แทนการวนลูปด้วย .run()
    }

    // ตรวจสอบ state ของโลกเกมหลังจากผ่านไป 3 เฟรม โดยตรงผ่าน World API
    let mut q = app.world_mut().query::<&Position>();
    for pos in q.iter(app.world()) {
        assert_eq!(pos.x, 3.0); // 0.0 + 1.0*3
        assert_eq!(pos.y, 6.0); // 0.0 + 2.0*3
    }
    println!("verified: position after 3 ticks matches expected value");
}
```

เราคอมไพล์และรันโค้ดนี้จริง — ได้ผลลัพธ์ตรงตาม assertion ทุกประการ (`x=3.0, y=6.0` หลังผ่านไป 3 เฟรม ตาม
สูตร `initial + velocity * ticks`) `app.world()`/`app.world_mut()` ให้ reference ไปยัง `World` ซึ่งเป็น
struct ที่เก็บ entity/component ทั้งหมดของเกมไว้จริง — เราสามารถสร้าง query ใหม่กลางทาง (นอกเหนือจาก system
ที่ลงทะเบียนไว้) ผ่าน `world.query::<T>()` เพื่อ "แอบดู" state ของเกมจากมุมมองของ test ได้โดยตรง นี่คือเทคนิค
เดียวกับที่ทีม Bevy ใช้เขียน unit test ของ `bevy_ecs` เองหลายพันเทสต์ในโปรเจกต์จริง

**สรุปข้อจำกัดของวิธีนี้อีกครั้งเพื่อความชัดเจน**: วิธีนี้ตรวจสอบได้แค่ "ตรรกะและ state ของ ECS" (ค่าตัวแปรใน
component/resource ถูกต้องตามที่คำนวณไว้หรือไม่) แต่ตรวจสอบ **ไม่ได้เลย** ว่าถ้ามีการ render จริง ภาพที่ออกมา
จะหน้าตาเป็นอย่างไร ถูกต้องตามตำแหน่ง/สีที่ตั้งใจไว้หรือไม่ — สองเรื่องนี้เป็นปัญหาคนละชั้นกัน (logic
correctness vs visual correctness) และบทนี้ตรวจสอบให้ได้เต็มที่แค่ชั้นแรกเท่านั้น

### 103.13 Bevy ในบริบทที่กว้างขึ้น: ตำแหน่งในโลก Game Dev ของ Rust

#### ทางเลือกอื่นในการทำเกมด้วย Rust (ระดับรู้จัก ไม่ลงรายละเอียด)

Bevy ไม่ใช่ทางเลือกเดียวสำหรับคนที่อยากทำเกมด้วย Rust แนวทางหลัก ๆ ที่มีอยู่ในระดับที่ควรรู้จัก (awareness
level เท่านั้น — บทนี้ไม่ได้พาเจาะลึกทางเลือกเหล่านี้):

1. **Godot ผ่าน `gdext`** — Godot เป็น game engine แบบ full-featured (มี editor แบบ visual, scene system,
   animation tools ครบ) ที่เขียนด้วย C++ แต่รองรับการเขียน game logic ด้วยภาษาอื่นผ่านระบบ GDExtension
   ชุมชน Rust มีโปรเจกต์ `gdext` (หรือชื่อ crate คือ `godot`) ที่เป็น binding ให้เขียนโค้ดเกมด้วย Rust แล้ว
   เรียกจาก Godot editor ได้ — เหมาะกับคนที่ต้องการ editor tooling ที่ครบเครื่องแบบ Godot แต่ยังอยากเขียน
   logic ด้วย Rust แทน GDScript/C#
2. **เขียน game logic เป็น Rust แล้วฝังใน engine ที่ไม่ใช่ Rust** — เช่น compile โค้ด Rust เป็น library
   (ผ่าน FFI ตามที่ **Part 43** สอนไว้) แล้วเรียกใช้จาก Unity/Unreal/engine อื่น ๆ เหมาะกับทีมที่มี pipeline
   เดิมอยู่แล้วแต่อยากได้ performance หรือความปลอดภัยของ Rust เฉพาะส่วน logic ที่สำคัญ
3. **Bevy** (ที่บทนี้สอน) — pure Rust ตั้งแต่ต้นจนจบ ไม่มี editor แบบ visual เต็มรูปแบบ (ยังพัฒนาอยู่ ณ
   ปัจจุบัน) แต่ได้ประโยชน์เต็มที่จากระบบ type ของ Rust, ECS ที่ integrate แน่นกับภาษา, และ community ที่ active
   มาก

#### จุดแข็งที่แท้จริงของ Bevy

- **Pure Rust ทั้ง stack** — ไม่มี FFI boundary ให้ต้องข้าม ไม่มีภาษาที่สองให้เรียนรู้ (ต่างจาก Godot ที่ยังต้อง
  รู้ GDScript หรือใช้ editor แยกภาษา) error handling, type system, ownership ทำงานสอดคล้องกันตลอดทั้งโปรเจกต์
- **ECS ที่ integrate เข้ากับ Rust's type system อย่างลึกซึ้ง** — อย่างที่เห็นในหัวข้อ 103.4 ว่า Bevy ใช้
  ข้อมูล type ของ `Query` พิสูจน์ความปลอดภัยของการรันคู่ขนานได้ตั้งแต่ compile/schedule-build time นี่คือสิ่งที่
  ทำได้ยากมากในภาษาที่ไม่มี ownership/borrow system แบบ Rust
- **Community ที่ active มาก** — release cycle ถี่ (หลาย version ต่อปี), examples เยอะ, ปลั๊กอินจากชุมชน
  (crates.io มี ecosystem ของ `bevy_*` third-party plugin จำนวนมาก)

#### จุดที่ต้องระวังจริง ๆ: API เปลี่ยนแปลงบ่อยระหว่าง version

นี่คือสิ่งที่บทนี้พยายามพิสูจน์ให้เห็นด้วยหลักฐานจริงตลอดทั้งบท ไม่ใช่แค่บอกลอย ๆ — เราตรวจสอบ source code
ข้าม version จริงและพบการเปลี่ยนแปลงที่ทำให้โค้ดเก่า **compile ไม่ผ่าน** กับเวอร์ชันใหม่อย่างน้อยดังนี้:

- **Event → Message** (หัวข้อ 103.6): `EventReader`/`EventWriter`/`add_event` (Bevy ≤0.16) กลายเป็น
  `MessageReader`/`MessageWriter`/`add_message` (Bevy ≥0.17) เพราะชื่อ `Event` ถูกเอาไปใช้กับระบบ Observer
  ใหม่แทน
- **Bundle struct → tuple + required components**: Bevy รุ่นเก่า (ก่อน ~0.11-0.15 ขึ้นกับ component) ต้อง
  สร้าง struct `Bundle` เอง (เช่น `Camera2dBundle { camera: .., transform: .., ... }`) ก่อนจะ spawn ได้ ส่วน
  Bevy ปัจจุบันใช้ tuple ธรรมดา + กลไก "required components" (`#[require(...)]`) แทนแล้วในหลายกรณี ทำให้โค้ด
  ตัวอย่างเก่าที่อ้าง `Camera2dBundle`/`SpriteBundle` ใช้ไม่ได้กับเวอร์ชันปัจจุบัน
- **`Input<T>` → `ButtonInput<T>`** (หัวข้อ 103.8): เปลี่ยนชื่อ resource ของ input ให้ชัดเจนขึ้น
- **Feature flag ของ crate `bevy` เอง** ก็ถูกจัดโครงสร้างใหม่หลายรอบ (เราเจอ `default_app`,
  `default_platform`, `2d_api`, `2d_bevy_render` ที่แยกละเอียดกว่า flat list แบบเดิมมาก ระหว่างตรวจสอบเนื้อหา
  บทนี้)

**บทเรียนที่ได้จากเรื่องนี้**: ถ้าคุณเจอ tutorial, บทความ, หรือ Stack Overflow answer เกี่ยวกับ Bevy ที่ไม่ได้
ระบุเวอร์ชันชัดเจน แล้วโค้ดตัวอย่างไม่ compile ให้สงสัยเรื่อง version mismatch เป็นอันดับแรก วิธีแก้ที่ดีที่สุด
คือดู **CHANGELOG อย่างเป็นทางการของ Bevy** (bevyengine.org มีหน้า "Migration Guides" แยกไว้ทุก major
release) และตรวจสอบเวอร์ชันที่ `Cargo.toml` ของโปรเจกต์คุณระบุไว้จริง ๆ ก่อนเชื่อโค้ดตัวอย่างจากที่ไหนก็ตาม
รวมถึงบทนี้ด้วย — ถ้าคุณอ่านบทนี้ในอนาคตที่ Bevy ออกเวอร์ชันใหม่กว่า 0.19.1 ไปแล้ว ให้เช็ค migration guide ก่อน
เสมอ เพราะเป็นไปได้สูงมากที่บางชื่อ type/method ในบทนี้จะถูกเปลี่ยนชื่อไปอีกแล้ว (Bevy เองก็ยังมี
0.20.0-rc.1 เป็น release candidate อยู่ระหว่างการเขียนบทนี้พอดี)

#### สรุปเปรียบเทียบแบบตรงไปตรงมา

| | Bevy | Godot + `gdext` | Rust logic + engine อื่น (FFI) |
|---|---|---|---|
| ภาษาที่ใช้ | Rust ล้วน | Rust (logic) + GDScript/C# (editor scripting ทั่วไป) | Rust (logic) + ภาษาของ engine หลัก |
| Visual editor | ยังไม่ครบเท่า engine ใหญ่ (พัฒนาต่อเนื่อง) | มีเต็มรูปแบบ (Godot editor) | มีเต็มรูปแบบ (ของ engine หลัก) |
| ประสิทธิภาพ ECS | สูงมาก (ออกแบบมาเพื่อสิ่งนี้ตั้งแต่แรก) | ปานกลาง-สูง (ไม่ใช่ ECS โดยธรรมชาติ) | ขึ้นกับ engine หลัก |
| ความเสี่ยงเรื่อง API เปลี่ยน | สูง (release cycle ถี่, breaking change บ่อยตามที่บทนี้แสดงให้เห็น) | ต่ำกว่า (Godot API นิ่งกว่า) | ต่ำ (engine หลักมักนิ่งกว่า) |
| เหมาะกับ | ทีมเล็ก/โปรเจกต์ที่อยากได้ Rust เต็ม stack, เกม indie, เกมที่เน้น performance ของ simulation | ทีมที่ต้องการ tooling ระดับ editor ครบแต่อยากเขียน logic ด้วย Rust | ทีมที่มี pipeline เดิมอยู่แล้วกับ engine ใหญ่ |

#### เชื่อมกลับไป Part 86: Bevy กับ WASM

**Part 86** เกริ่นไว้แล้วว่า Bevy compile เป็น WASM ได้ และเกมที่เขียนด้วย Bevy รันในเบราว์เซอร์ได้ตรง ๆ โดยไม่
ต้องติดตั้งอะไรเพิ่ม — ตอนตรวจสอบเนื้อหาบทนี้เราเจอหลักฐานที่ยืนยันเรื่องนี้ในระดับ `Cargo.toml` ของ crate
`bevy` เองจริง ๆ: มี feature flag ชื่อ `webgl2` และ `webgpu` อยู่ในรายการ feature ทั้งหมด (ที่เราเห็นตอน
`cargo add bevy --features dynamic_linking --dry-run` แสดงรายการ feature ที่เป็นไปได้ทั้งหมดออกมา) ซึ่งเป็น
backend การ render สองแบบที่ `wgpu` (ตัว renderer ที่ Bevy ใช้) รองรับสำหรับรันในเบราว์เซอร์โดยเฉพาะ (`webgl2`
ใช้ WebGL 2.0 ที่เบราว์เซอร์รุ่นเก่ากว่ารองรับได้กว้างกว่า, `webgpu` ใช้ WebGPU มาตรฐานใหม่กว่าที่เร็วกว่าแต่
เบราว์เซอร์รองรับน้อยกว่า) — นี่ยืนยันว่าคำกล่าวใน Part 86 ไม่ใช่แค่ทฤษฎี แต่มี feature flag รองรับอยู่จริงใน
โครงสร้าง dependency ของ Bevy ปัจจุบัน แม้บทนี้จะไม่ได้ลงมือ compile เป็น WASM จริงเพื่อตรวจสอบ (นอกขอบเขตที่
กำหนดไว้ของบทนี้) ก็ตาม

#### บทนี้คือจุดเริ่มต้น ไม่ใช่ความเชี่ยวชาญ

Bevy เป็น engine ที่ใหญ่มาก — บทนี้ครอบคลุมแค่ ECS fundamentals (entity/component/system/resource/message),
schedule (`Startup`/`Update`/`FixedUpdate`), การจัดโครงสร้างโค้ดด้วย `Plugin`, การจัดการสถานะด้วย `States`,
และการ render/input ระดับพื้นฐานที่สุดเท่านั้น หัวข้อที่ยังไม่ได้พูดถึงเลยและมีความสำคัญมากถ้าจะทำเกมจริงจัง
ได้แก่: asset loading แบบเต็มรูปแบบ (texture atlas, audio, font), animation system, physics (Bevy ไม่มี
physics engine ในตัว ต้องพึ่ง crate ภายนอกเช่น `avian` หรือ `rapier`), UI system เต็มรูปแบบ (Bevy มี
`bevy_ui`/feature `bevy_feathers` ที่เราเห็นชื่อผ่านมาระหว่างตรวจสอบ แต่ไม่ได้ลงรายละเอียด), scene
serialization, networking multiplayer, และการ optimize สำหรับ target ที่หลากหลาย (mobile, WASM ตามที่อธิบาย
ข้างบน, console) ผู้อ่านที่สนใจต่อควรไปที่ **Bevy's official examples repository** (มีตัวอย่างเป็นร้อยไฟล์
ครอบคลุมทุกฟีเจอร์) และ **Bevy's official book/cheatbook** ของชุมชนเป็นจุดเริ่มต้นถัดไป

## กับดักที่พบบ่อย (Common Pitfalls)

กับดักทั้ง 7 ข้อด้านล่างนี้ทุกข้อผ่านการทดสอบจริงระหว่างตรวจสอบเนื้อหาบทนี้ — เราตั้งใจเขียนโค้ดที่ผิดแต่ละแบบ
ขึ้นมาจริง แล้ว compile/run เพื่อ capture ข้อความ error ตัวจริงจาก compiler หรือ runtime panic ของ Bevy 0.19.1
มาแสดงให้เห็น (ไม่ใช่ error message ที่เดาหรือจำจากความรู้ทั่วไป) สังเกตว่ากับดักครึ่งแรก (ข้อ 1, 4, 5, 7) เป็น
เรื่อง type system ที่ compiler จับได้ตั้งแต่ก่อนรันโปรแกรม ส่วนครึ่งหลัง (ข้อ 2, 3) เป็นเรื่องที่ compiler จับ
ไม่ได้ (เพราะ Bevy ตรวจสอบความถูกต้องบางอย่าง เช่นการลงทะเบียน message type หรือความขัดแย้งของ query เอาไว้ที่
runtime/schedule-build-time แทน ไม่ใช่ compile time เต็มรูปแบบ) — เป็นตัวอย่างที่ดีว่าถึงแม้ Rust's type system
จะช่วยจับ bug ได้มากแค่ไหน ก็ยังมีบางเรื่องที่ ECS framework ต้องตรวจสอบเองตอน runtime อยู่ดี เพราะข้อมูลบางอย่าง
(เช่น "message type นี้ถูกลงทะเบียนไว้หรือยัง") ไม่ได้อยู่ใน type system ของ Rust โดยตรง

**1. ลืม `#[derive(Component)]` แล้ว spawn struct นั้นตรง ๆ**

```rust,ignore
struct Position { x: f32, y: f32 } // ลืม derive!

fn setup(mut commands: Commands) {
    commands.spawn(Position { x: 0.0, y: 0.0 });
}
```

Error จริงจาก compiler:

```
error[E0277]: the trait bound `Position: Component` is not satisfied
  --> src/main.rs:6:20
   |
 6 |     commands.spawn(Position { x: 0.0, y: 0.0 });
   |              ----- ^^^^^^^^^^^^^^^^^^^^^^^^^^^ invalid `Bundle`
   |
   = help: the trait `Component` is not implemented for `Position`
   = note: consider annotating `Position` with `#[derive(Component)]` or `#[derive(Bundle)]`
```

วิธีแก้: เติม `#[derive(Component)]` เหนือ struct เสมอ ถ้าตั้งใจจะแนบ struct นี้เข้ากับ entity เดี่ยว ๆ (ถ้า
ตั้งใจจะรวมหลาย component เข้าเป็นกลุ่มไว้ spawn พร้อมกันบ่อย ๆ ให้ใช้ `#[derive(Bundle)]` กับ struct ที่มี
field เป็น component แต่ละตัวแทน)

**2. ใช้ `Message`/`MessageReader`/`MessageWriter` โดยไม่ได้ลงทะเบียนด้วย `app.add_message::<T>()`**

```rust,ignore
#[derive(Message)]
struct ScoreEvent;

fn reader_system(mut reader: MessageReader<ScoreEvent>) {
    for _ in reader.read() { println!("got score event"); }
}

fn main() {
    App::new()
        .add_plugins(MinimalPlugins)
        // ลืม .add_message::<ScoreEvent>() !
        .add_systems(Update, reader_system)
        .run();
}
```

โค้ดนี้ **compile ผ่าน** แต่ panic ตอน runtime ทันทีที่เฟรมแรกรัน ด้วย error จริงจากการทดสอบ:

```
thread 'Compute Task Pool (0)' panicked at .../bevy_ecs-0.19.1/src/error/handler.rs:130:1:
Encountered an error in system `<Enable the debug feature to see the name>`:
Parameter `<Enable the debug feature to see the name>::messages` failed validation:
Message not initialized
If this is an expected state, wrap the parameter in `Option<T>` and handle `None`
when it happens, or wrap the parameter in `If<T>` to skip the system when it happens.
```

วิธีแก้: เพิ่ม `.add_message::<ScoreEvent>()` เข้าไปในสาย builder ของ `App` ก่อนที่จะมี system ไหนใช้
`MessageReader<ScoreEvent>`/`MessageWriter<ScoreEvent>` — จำง่าย ๆ ว่า "ประกาศชนิด message ก่อนใช้งานเสมอ"
เหมือนกับต้อง `init_resource`/`insert_resource` ก่อนจะ `Res`/`ResMut` ได้

**3. เขียน `Query` สองตัวในระบบเดียวกันที่เข้าถึง component เดียวกันแบบ mutable ทับซ้อนกัน (ไม่มี
`With`/`Without` แยก)**

```rust,ignore
fn conflicting_system(
    mut players: Query<&mut Position, With<Player>>,
    mut all: Query<&mut Position>, // ทับซ้อนกับตัวบน! อาจมี entity ที่เข้าทั้งคู่
) {
    for mut p in &mut players { p.x += 1.0; }
    for mut p in &mut all { p.x += 1.0; }
}
```

โค้ดนี้ compile ผ่าน แต่ panic ตอน runtime ทันทีที่ schedule ถูก initialize ด้วย error B0001 จริง:

```
thread 'main' panicked at .../bevy_ecs-0.19.1/src/query/state.rs:216:13:
error[B0001]: Query<&mut Position, With<Player>> in system conflicting_system
accesses component(s) Position in a way that conflicts with a previous system
parameter. Consider using `Without<T>` to create disjoint Queries or merging
conflicting Queries into a `ParamSet`. See: https://bevy.org/learn/errors/b0001
```

วิธีแก้ตามที่ error message บอกไว้ตรง ๆ: ใช้ `Without<T>` ทำให้สอง query แยกกันเด็ดขาดในระดับ type (เช่น
เปลี่ยน query ที่สองเป็น `Query<&mut Position, Without<Player>>`) หรือถ้าจำเป็นต้องเข้าถึง component ชุด
เดียวกันแบบซ้อนทับจริง ๆ (เช่นตั้งใจให้ query แรกแก้ไข "เฉพาะผู้เล่น" และ query ที่สองแก้ไข "ทุกคนรวมผู้เล่น
ด้วย" ซึ่งไม่มีทางใช้ `Without` แยกได้เพราะมันตั้งใจให้ทับกันจริง ๆ) ให้ใช้ `ParamSet` ซึ่งบอก Bevy อย่างชัดเจน
ว่า "ฉันรับรองว่าจะไม่เข้าถึงสอง query นี้พร้อมกันในโค้ดของฉันเอง ให้ฉันสลับใช้ทีละตัวได้":

```rust
use bevy::prelude::*;

# #[derive(Component)]
# struct Position { x: f32 }
# #[derive(Component)]
# struct Player;
#
fn param_set_system(
    mut set: ParamSet<(Query<&mut Position, With<Player>>, Query<&mut Position>)>,
) {
    for mut p in set.p0().iter_mut() {
        p.x += 1.0; // เข้าถึงผ่าน .p0() ก่อน
    }
    for mut p in set.p1().iter_mut() {
        p.x += 100.0; // แล้วเข้าถึงผ่าน .p1() ทีหลัง คนละช่วงเวลากัน ไม่ทับกันจริง
    }
}
```

เราคอมไพล์และรันโค้ดนี้จริง — ไม่มี panic B0001 เหมือนก่อนหน้า เพราะ `ParamSet` บอก Bevy ว่า "สอง query นี้จะไม่
ถูกยืมออกมาพร้อมกัน" ทำให้ Bevy ยอมให้ query ที่ทับกันในระดับ component อยู่ในระบบเดียวกันได้ โดยแลกกับการที่
เราต้องเรียก `.p0()`/`.p1()` แทนการใช้ตัวแปรตรง ๆ (ข้อแลกเปลี่ยน: `ParamSet` ทำให้ compiler ช่วยตรวจสอบความ
ปลอดภัยของการยืมได้น้อยลงกว่าการแยก query ด้วย `Without` ตั้งแต่แรก — ควรใช้ `Without` เป็นตัวเลือกแรกเสมอถ้า
ทำได้ และใช้ `ParamSet` เฉพาะกรณีที่ตั้งใจให้ query ทับกันจริง ๆ เท่านั้น)

**4. ใช้ `Res<T>` แล้วพยายามแก้ไขค่าข้างในตรง ๆ (ต้องใช้ `ResMut<T>`)**

```rust,ignore
fn bump_score(score: Res<Score>) {
    score.0 += 1; // Res คืออ่านอย่างเดียว!
}
```

Error จริงจาก compiler:

```
error[E0594]: cannot assign to data in dereference of `Res<'_, Score>`
 --> src/main.rs:7:5
  |
7 |     score.0 += 1;
  |     ^^^^^^^^^^^^ cannot assign
  |
  = help: trait `DerefMut` is required to modify through a dereference,
    but it is not implemented for `Res<'_, Score>`
```

วิธีแก้: เปลี่ยน parameter type เป็น `ResMut<Score>` — จำคู่กับ `&T`/`&mut T` ธรรมดาที่ **Part 7** สอนไว้:
`Res<T>` คือ `&T` เชิงแนวคิด, `ResMut<T>` คือ `&mut T` เชิงแนวคิด

**5. ใส่ filter (`With<T>`/`Without<T>`) ปนเข้าไปใน tuple ข้อมูลตัวแรกของ `Query` ผิดตำแหน่ง**

```rust,ignore
// ผิด: Without<Obstacle> ปนอยู่ใน "data" tuple ตัวแรก ไม่ใช่ "filter" ตัวที่สอง
fn wrong_query(query: Query<(&Position, Without<Obstacle>)>) {
    for (_pos, _) in &query {}
}
```

Error จริงจาก compiler:

```
error[E0277]: `Without<Obstacle>` is not valid to request as data in a `Query`
 --> src/main.rs:9:23
  |
9 | fn wrong_query(query: Query<(&Position, Without<Obstacle>)>) {
  |                       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ invalid `Query` data
  |
  = help: the trait `QueryData` is not implemented for `Without<Obstacle>`
  = note: if `Without<Obstacle>` is a component type, try using
    `&Without<Obstacle>` or `&mut Without<Obstacle>`
```

วิธีแก้: ย้าย filter ไปเป็น **type parameter ตัวที่สอง** ของ `Query` เสมอ — `Query<&Position,
Without<Obstacle>>` (ไม่ใช่ปนอยู่ในวงเล็บเดียวกับ data) จำโครงสร้างให้แม่น: `Query<Data, Filter>` — ตัวแรกคือ
"จะเอาข้อมูลอะไรออกมา" ตัวที่สองคือ "จะกรอง entity ด้วยเงื่อนไขอะไร"

**6. ลืมว่า `dynamic_linking` เป็นแค่ dev-time optimization แล้วใส่ทิ้งไว้ตอน build release**

ไม่มี compiler error ให้เห็น — โปรแกรมจะ build ผ่านและรันได้ปกติตอน dev แต่ถ้าคุณลืมปิด feature นี้ตอน
`cargo build --release` แล้ว **แจกจ่ายเฉพาะไฟล์ binary** ให้คนอื่นโดยไม่แถมไฟล์ `.so`/`.dll` ของ Bevy ไปด้วย
ผู้ใช้ปลายทางจะเจอ error ตอนเปิดโปรแกรม (บน Linux เช่น `error while loading shared libraries:
libbevy_dylib....so: cannot open shared object file: No such file or directory`) — วิธีป้องกัน: อย่าใส่
`dynamic_linking` ใน `[dependencies]` ของ `Cargo.toml` แบบถาวร ให้เปิดมันผ่าน command line flag
(`cargo run --features bevy/dynamic_linking`) เฉพาะตอน dev เท่านั้น แล้วตรวจสอบก่อน release เสมอว่า
`cargo build --release` ไม่มี feature นี้ติดไปด้วย

**7. ปิด `default-features` ของ `bevy` เพื่อลด compile time (ตามหัวข้อ 103.7) แล้วลืมว่า type ที่ต้องใช้หายไป
ด้วย**

```rust,ignore
// Cargo.toml: bevy = { version = "0.19.1", default-features = false,
//                       features = ["default_app", "multi_threaded"] }
// (ไม่มี feature "2d"/"2d_api" ที่ให้ type Camera2d)
use bevy::prelude::*;

fn setup(mut commands: Commands) {
    commands.spawn(Camera2d);
}
```

Error จริงจาก compiler:

```
error[E0425]: cannot find value `Camera2d` in this scope
 --> src/main.rs:4:20
  |
4 |     commands.spawn(Camera2d);
  |                    ^^^^^^^^ not found in this scope
```

ข้อความ error แบบนี้ (`cannot find value/type ... in this scope`) มักทำให้เข้าใจผิดว่าลืม `use` หรือพิมพ์ชื่อ
ผิด แต่ถ้าคุณแน่ใจว่าเขียนชื่อถูกและ `use bevy::prelude::*;` ไว้แล้ว สาเหตุที่พบบ่อยที่สุดคือ **feature ของ
crate `bevy` ที่เปิดไว้ใน `Cargo.toml` ไม่ครอบคลุม type นั้น** (เพราะปิด `default-features` ไว้เพื่อลดเวลา
compile ตามที่หัวข้อ 103.7 สอน) วิธีแก้: เพิ่ม feature ที่มี type นั้นกลับเข้าไป (สำหรับ `Camera2d`/`Sprite`
คือ feature `2d` เต็มรูปแบบถ้าต้องการ render จริง หรือ `2d_api` ถ้าต้องการแค่ชนิดข้อมูลแบบในหัวข้อ 103.7) หรือ
ถ้าไม่ได้ตั้งใจปิด default features เพื่อ optimize อะไรเป็นพิเศษ ทางที่ปลอดภัยที่สุดคือใช้
`bevy = "0.19.1"` เฉย ๆ (เปิด default features ตามปกติ) แล้วค่อยมาปิดทีละส่วนตอนที่เข้าใจ feature graph ของ
Bevy ดีขึ้นแล้ว

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เขียน component `Health { current: i32, max: i32 }` และ system `regen_system` ที่เพิ่ม
   `current` ขึ้น 1 หน่วยทุกเฟรม แต่ไม่ให้เกิน `max` (ใช้ `Query<&mut Health>`) ทดสอบด้วยการ spawn entity
   ที่มี `Health { current: 50, max: 100 }` แล้วรัน `app.update()` แบบ headless (ใช้ `MinimalPlugins`
   เหมือนตัวอย่างในบท) 60 ครั้ง แล้ว assert ว่า `current == 100` (ไม่ใช่ 110) โจทย์นี้ฝึกพื้นฐานที่สุดของบท:
   เขียน component เอง, เขียน system ที่ query แบบ mutable, และตรวจสอบผลลัพธ์แบบ headless ด้วย
   `app.world_mut().query::<&Health>()` ตามที่หัวข้อ 103.12 สอนไว้
   *Hint*: ใช้ `.min(max)` หรือ `if` เช็คก่อนบวก — ระวังโจทย์แบบนี้เป็นกับดักคลาสสิกที่มักลืมเช็คเพดานแล้วปล่อยให้
   ค่าทะลุ `max` ไปเรื่อย ๆ

2. **(กลาง)** ขยายมินิเกมจากหัวข้อ 103.11 ให้มีสิ่งกีดขวางได้ **หลายตัวพร้อมกัน** (spawn 3 obstacle ที่ตำแหน่ง
   x ต่างกันใน `setup`) และเพิ่ม system ใหม่ที่ despawn (ลบ) obstacle ที่ตกต่ำกว่า `y = -10` ไปแล้ว (ใช้
   `commands.entity(entity).despawn()` — ต้อง query แบบที่ได้ `Entity` ID มาด้วย ไม่ใช่แค่ component) โจทย์นี้
   ฝึกการจัดการ entity หลายตัวที่มีชนิดเดียวกันพร้อมกัน (ต่างจากตัวอย่างในบทที่มี obstacle แค่ตัวเดียวเพื่อความ
   ง่าย) และฝึกการลบ entity ผ่าน `Commands` ซึ่งบทนี้ยังไม่ได้สอนตรง ๆ มาก่อน
   *Hint*: `Query<(Entity, &Position), With<Obstacle>>` จะให้ทั้ง `Entity` ID และ component ในลูปเดียวกัน —
   ระวังอย่าเรียก `despawn()` ขณะกำลังวน loop บน `Query` ตัวเดียวกันตรง ๆ (เพราะ `Commands` defer การเปลี่ยน
   โครงสร้างไว้ก่อนตามที่หัวข้อ 103.2 อธิบาย จึงปลอดภัยที่จะเรียกกลางลูปได้โดยไม่ error แต่การเปลี่ยนแปลงจะ
   เกิดขึ้นจริงตอนจบเฟรมเท่านั้น ไม่ใช่ทันที)

3. **(ยาก)** เพิ่มระบบ "ด่าน" ให้มินิเกม: ใช้ `Resource` ใหม่ชื่อ `Wave(u32)` เก็บว่าอยู่ด่านที่เท่าไหร่ เมื่อ
   `Score` ถึงเกณฑ์ที่กำหนด (เช่นทุก ๆ 50 แต้ม) ให้เพิ่ม `Wave` ขึ้น 1 และ spawn obstacle ใหม่เพิ่มเข้ามาอีก 1
   ตัวที่ตำแหน่งสุ่ม (ใช้ message ชนิดใหม่ `WaveUpEvent` ส่งสัญญาณระหว่าง `scoring_system` กับ system ที่
   spawn obstacle ใหม่ เพื่อรักษาการ decouple ตามหัวข้อ 103.6) ทดสอบแบบ headless ว่าหลังจากคะแนนถึง 50 แล้ว
   จำนวน entity ที่มี `Obstacle` component เพิ่มขึ้นจริง โจทย์นี้ฝึกการออกแบบ message ของตัวเองตั้งแต่ต้น
   (ไม่ใช่แค่ใช้ตามตัวอย่างในบท) และฝึกผสาน resource ใหม่ (`Wave`) เข้ากับ resource เดิม (`Score`) โดยไม่ทำให้
   ระบบเดิมพัง
   *Hint*: การสุ่มตำแหน่งต้องใช้ crate `rand` (จาก **Part 13/54** ที่เคยใช้) ไม่ใช่ของ Bevy เอง — Bevy ไม่มี
   random number generator ของตัวเองแบบ built-in ใน core ECS (มี crate เสริมชื่อ `bevy_rand` จากชุมชนถ้าอยาก
   ได้ random ที่ integrate กับ ECS โดยตรง แต่โจทย์นี้ใช้ `rand` ธรรมดาก็เพียงพอ)

4. **(ประยุกต์ใช้งานจริง)** สร้างระบบ "high score" ที่ใช้ `Resource` เก็บคะแนนสูงสุดที่เคยทำได้ (คงอยู่แม้เกม
   จบแล้วเริ่มใหม่ — ต้องแยก `Resource` ตัวนี้ออกจาก `Score` ที่ reset ทุกครั้งเริ่มเกมใหม่) และเขียน
   `MessageReader`/`MessageWriter` คู่ใหม่ชื่อ `GameRestartEvent` ที่เมื่อได้รับ (จำลองด้วยการกด `KeyCode::KeyR`
   ตอน `GameState::GameOver`) จะ reset `Score` และ entity ทั้งหมดกลับไปที่ตำแหน่งเริ่มต้น แต่ **ไม่แก้ไข**
   high score resource เลย เขียน headless test จำลองว่าเล่นจบเกมสองรอบ (คะแนนรอบแรกน้อยกว่ารอบสอง) แล้ว
   assert ว่า high score หลังจบรอบสองตรงกับคะแนนรอบสองพอดี (ไม่ใช่รอบแรก) โจทย์นี้รวมทุกอย่างที่บทนี้สอนเข้า
   ด้วยกัน (component, resource หลายตัวที่มี lifetime ต่างกัน, message, state transition) และใกล้เคียงกับ
   ฟีเจอร์ที่เกมจริงส่วนใหญ่ต้องมี
   *Hint*: logic การเปรียบเทียบ "ถ้าคะแนนใหม่มากกว่า high score เดิม ให้อัปเดต" ควรอยู่ใน system ที่ทำงานตอน
   `game_over_system` เปลี่ยน state เป็น `GameOver` (เชื่อมกับ message เดิมที่มีอยู่แล้วได้ ไม่ต้องสร้าง
   message ใหม่สำหรับส่วนนี้) — ลองพิจารณาด้วยว่าถ้าใช้ `States`/`OnEnter` เต็มรูปแบบตามหัวข้อ 103.10 แทน
   `enum` ที่เก็บใน `Resource` ธรรมดา โค้ดจะดูสะอาดขึ้นหรือไม่อย่างไร

## สรุป

บทนี้พาไปรู้จัก Bevy game engine ผ่านมุมมองที่ต่างจากทุกบทก่อนหน้าในหลักสูตรนี้อย่างสิ้นเชิง: สถาปัตยกรรม
Entity Component System (ECS) ซึ่งแก้ปัญหาคลาสสิกของ OOP inheritance ในเกม (ปัญหา "ศัตรูที่บินได้และว่ายน้ำ
ได้ควรสืบทอดจากไหน") ด้วยการแยกข้อมูล (Component) ออกจากพฤติกรรม (System) โดยสิ้นเชิง แล้วผูกทั้งสองเข้าด้วยกัน
ผ่าน ID เปล่า ๆ (Entity) — เป็นการนำหลักการ composition-over-inheritance ที่ **Part 19** เกริ่นไว้ไปใช้ในระดับ
สถาปัตยกรรมทั้งเอนจิน

เราเรียนรู้การติดตั้ง Bevy จริง (`cargo add bevy`, เวอร์ชัน 0.19.1 ที่ตรวจสอบแล้ว), การเขียนและอ่าน `Query`
หลายรูปแบบ (`&T`, `(&mut T, &U)`, `With<T>`/`Without<T>`), การใช้ `Resource` สำหรับข้อมูล global, และระบบ
`Message` (ที่เพิ่งเปลี่ยนชื่อมาจาก `Event` ใน Bevy 0.17 — ตัวอย่างที่เป็นรูปธรรมที่สุดของหัวข้อ 103.13 ที่บอก
ว่า Bevy's API เปลี่ยนแปลงบ่อย) สำหรับการสื่อสารแบบ decoupled ระหว่าง system ซึ่งเชื่อมโยงกับแนวคิด message
passing ที่ **Part 38** สอนไว้ในบริบทของ OS thread

ที่สำคัญที่สุด บทนี้พยายามซื่อสัตย์เต็มที่กับสิ่งที่ตรวจสอบได้จริงในสภาพแวดล้อม headless: ทุกตัวอย่างโค้ด ECS
logic (movement, resource, message, input, มินิเกมสมบูรณ์) ถูกคอมไพล์และรันจริงพร้อม assertion ที่ผ่านทั้งหมด
แต่ "ภาพที่ render จริงบนจอ" ไม่สามารถตรวจสอบได้เลยในสภาพแวดล้อมที่ไม่มี GPU/display — ข้อจำกัดนี้ต่างจากบท
web/WASM ก่อนหน้าอย่างสิ้นเชิง เพราะ Bevy render ผ่าน GPU โดยตรง ไม่มี "headless browser" เทียบเท่าให้ใช้

บทนี้เป็นแค่จุดเริ่มต้นของการทำเกมด้วย Bevy เท่านั้น — ยังมีอีกมากที่ไม่ได้พูดถึง (physics, animation, UI
system, asset pipeline เต็มรูปแบบ, multiplayer) ผู้อ่านที่สนใจต่อควรไปอ่าน official examples และ book ของ
Bevy โดยตรง และถ้าสนใจ deployment เป็นเว็บแอปสามารถย้อนกลับไปดู **Part 86-87** เรื่อง WASM ได้ เพราะ Bevy
รองรับ WASM เป็น deployment target หนึ่งที่ใช้งานได้จริง

Part ถัดไป (**Part 104**) จะพาไปสำรวจโดเมนที่ต่างออกไปอีกขั้ว — Blockchain และ Smart Contracts ด้วย Rust ซึ่ง
เป็นอีกตัวอย่างของการที่ Rust ถูกเลือกใช้ในโดเมนที่ต้องการความปลอดภัยของ memory และความถูกต้องของ logic ในระดับ
สูงมาก แม้จะเป็นโดเมนที่ต่างจากเกมโดยสิ้นเชิงก็ตาม

---

**Part ก่อนหน้า:** [Embedded Rust เบื้องต้น](part-102-embedded-rust.md) | **Part ถัดไป:** [Blockchain และ Smart Contracts ด้วย Rust](part-104-blockchain-smart-contracts.md)
