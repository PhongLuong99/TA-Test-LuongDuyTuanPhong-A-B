# Technical Breakdown - Character Animation Tech (TestB)

## 1. Implementation Approach

### 1.1 Trạng thái gameplay và trạng thái animation

Character Blueprint sở hữu các trạng thái phục vụ gameplay:

- `WeaponStatus` kiểu `Name` xác định loại vũ khí đang được chọn.
- `Sword`, `GreatSword`, `Axe` lưu tham chiếu đến các weapon instance.
- `isWeaponSwitching` ngăn một equip transition mới bắt đầu trong khi montage hiện tại đang chạy.

Animation Blueprint sử dụng một trạng thái có kiểu rõ ràng là `E_CombatType`, gồm `None`, `Sword`, `GreatSword`, `Axe`. Khi vũ khí thay đổi, hàm `BP_ThirdPersonCharacter::Update Animation State` ánh xạ `WeaponStatus` sang `E_CombatType`, sau đó gửi giá trị này đến Anim Instance hiện tại thông qua interface `BPI_Animations`.

![Enum combat type đã triển khai](Media/Screenshots_Test_B/Technical_Detail4.png)

AnimBP chỉ nhận dữ liệu cần thiết để tạo pose. Interface cũng giúp tránh cast trực tiếp sang `ABP_Manny`, nhờ đó Character ít phụ thuộc vào một Anim Blueprint cụ thể hơn.

### 1.2 Vòng đời weapon actor

Prototype sử dụng phương án destroy-and-respawn khi chuyển vũ khí giữa trạng thái equipped và world.

**Quy trình drop**

1. Destroy actor vũ khí đang được Character tham chiếu.
2. Lấy class của vũ khí và spawn một actor thay thế tại transform của Character.
3. Gọi `Drop Weapon` qua `BPI_Interactions` trên instance mới để bật hành vi của vũ khí trong world.

**Quy trình pickup**

1. Chạy `Sphere Trace For Objects` với object channel `Weapon` 
2. Lưu vũ khí trúng trace thông qua interaction interface.
3. Drop vũ khí đang cầm dựa trên `WeaponStatus`.
4. Spawn actor mới bằng class của vũ khí vừa phát hiện.
5. Gán instance đó vào weapon reference phù hợp và gọi `EquipWeapon`.

### 1.3 Equip và chuyển vũ khí

Mỗi input equip kiểm tra tính hợp lệ của weapon reference, từ chối transition mới nếu `isWeaponSwitching` đang bật, sau đó khóa trạng thái và chọn montage theo `WeaponStatus` hiện tại. Trên nhánh hoàn tất bình thường `On Completed`, graph ghi trạng thái mới, mở khóa và cập nhật AnimBP.

### 1.4 Chuyển socket bằng Anim Notify

Việc attach vũ khí được đồng bộ tới đúng frame của swap montage thông qua ba Anim Notify Blueprint: `AN_AttachSword`, `AN_AttachGreatSword`, `AN_AttachAxe`.

Mỗi notify thực hiện cùng một chuỗi có kiểm tra:

1. Lấy owner của `Mesh Comp` đang chạy montage và cast sang `BP_ThirdPersonCharacter`.
2. Đọc weapon reference tương ứng và kiểm tra `Is Valid`.
3. Dùng Boolean `AttachToHand` của notify làm index cho node `Select`.
4. Chọn `AttachToSocket` khi false hoặc `HandSocket` khi true.
5. Gọi `Attach to Character` với socket vừa chọn.

### 1.5 Lựa chọn animation

`ABP_Manny` giữ việc lựa chọn locomotion bên trong state machine `Locomotion`:

- State `Idle` chọn `MM_Idle`, `a_Sword_Idle`, `a_GreatSwordIdle` hoặc `a_Axe_Idle` bằng `Blend Poses (E_CombatType)`.
- State `Walk / Run` đưa `Ground Speed` vào các Blend Space Player của tay không và từng loại vũ khí, sau đó chọn kết quả bằng cùng enum.
- Blend time riêng cho từng pose giúp chuyển đổi hình ảnh mượt hơn sau khi đổi vũ khí.

## 2. UE-specific Workflows

### Enhanced Input

Enhanced Input Actions điều khiển thao tác tương tác và chọn vũ khí. Graph cho thấy `IA_Interact` và các equip action riêng cho từng category. Input được xử lý trong `BP_ThirdPersonCharacter`, nơi điều phối reference, montage, weapon actor và trạng thái animation.

### Blueprint Interfaces

Hai interface boundary có thể quan sát được:

- `BPI_Interactions` giao tiếp với weapon actor để lưu interactable và đưa vũ khí vào trạng thái dropped.
- `BPI_Animations` cập nhật combat type của Anim Instance mà không cần hard cast.

Interface giúp bên gọi không phụ thuộc vào một weapon child hoặc AnimBP cụ thể, miễn là receiver thực hiện đúng contract.

`BP_WeaponBase` triển khai ba interaction event có thể quan sát được. `StoreTempWeapon` ghi weapon vào reference `Temp Weapon` của Character. `InteractWithWeapon` tắt simulation trên skeletal mesh, truyền weapon sang Character dưới dạng `Pick Up Weapon`, rồi destroy world actor. `DropWeapon` bật lại skeletal-mesh simulation cho dropped instance.

### Collision, physics và attachment

`EquipWeapon` thực hiện các thao tác sau trên weapon instance mới:

1. Đặt collision profile thành `NoCollision`.
2. Gọi `Attach to Character` bằng giá trị `Attach to Socket` cấu hình trong weapon.
3. Tắt physics simulation trên skeletal mesh component được thể hiện trong graph.

`BP_BaseItem::AttachToCharacter` lấy Character đang sở hữu item, dùng Character Mesh làm parent component, attach weapon actor vào socket được truyền vào và áp dụng `Snap to Target` cho location, rotation, scale.

### Layering trong Animation Blueprint

Final pose graph kết hợp:

1. Cached pose từ locomotion và main state.
2. Slot `UpperBody`.
3. `Layered Blend per Bone` để đặt upper-body action lên locomotion.
4. Montage path qua `DefaultSlot`.
5. Control Rig node trước output pose.

## 3. Code / Blueprint Architecture
### 3.1 Trách nhiệm của các Blueprint chính

| Blueprint / asset | Trách nhiệm |
|---|---|
| `BP_ThirdPersonCharacter` | Input, trace, trạng thái vũ khí hiện tại, actor reference, điều phối swap montage, drop/pickup và cập nhật AnimBP. |
| `BP_BaseItem` | Các item component dùng chung và hàm attach vào Character Mesh. |
| `BP_Weapon_Base` và các weapon BP con | Mesh/default data, combat type, `AttachToSocket`, `HandSocket` và interaction-interface behavior. |
| `BPI_Interactions` | Interaction contract giữa Character và weapon actor. |
| `BPI_Animations` | Combat-type update contract giữa Character và Anim Instance. |
| `E_CombatType` | Animation category có kiểu rõ ràng: `None`, `Sword`, `GreatSword`, `Axe`. |
| `ABP_Manny` | Movement state, lựa chọn weapon pose, montage layering và Control Rig pass cuối. |
| `AN_AttachSword`, `AN_AttachGreatSword`, `AN_AttachAxe` | Chuyển weapon giữa socket lưu trữ và socket tay đúng thời điểm trong montage. |

### 3.1 Quan hệ giữa các system

```text
Enhanced Input
      │
      ▼
BP_ThirdPersonCharacter
  ├─ interaction trace / điều phối pickup
  ├─ equipped actor references
  ├─ WeaponStatus (Name)
  ├─ chọn montage + transition lock
  ├─ DropSword / DropGreatSword / DropAxe
  └─ Update Animation State
       │
       ├──────── BPI_Animations ───────► ABP_Manny
       │                                  ├─ E_CombatType
       │                                  ├─ Locomotion state machine
       │                                  ├─ lựa chọn weapon pose
       │                                  ├─ montage slots / per-bone blend
       │                                  └─ Control Rig node
       │
       └──────── BPI_Interactions ─────► BP_BaseItem / weapon children
                                          ├─ collision components
                                          ├─ skeletal/static mesh components
                                          ├─ dữ liệu AttachToSocket / HandSocket
                                          ├─ AttachToCharacter (Snap to Target)
                                          ├─ EquipWeapon path
                                          └─ Drop Weapon behavior

Swap Montages ─► AN_AttachSword / GreatSword / Axe
                    └─ chọn socket lưu trữ hoặc socket tay tại notify frame
```

## 4. Problem-solving Process

### Đồng bộ gameplay và animation

**Thách thức:** Một lần đổi vũ khí ảnh hưởng đồng thời tới actor state, montage state, locomotion pose và Anim Instance.

**Giải pháp hiện tại:** Các nhánh equip hoàn tất bình thường cùng hội tụ về `Update Animation State`, nơi trạng thái gameplay được chuyển đổi và gửi sang AnimBP qua interface.

**Cách kiểm tra:** Test mọi cặp chuyển đổi cũ → mới, bao gồm None → weapon, weapon → weapon và weapon → None. Sau montage, cần xác nhận equipped actor, `WeaponStatus`, `E_CombatType` và final pose thống nhất với nhau.

### Ngăn các weapon swap chồng lên nhau

**Thách thức:** Input equip liên tục có thể khởi chạy nhiều montage và tạo ra các lần ghi trạng thái xung đột.

### Đồng bộ weapon actor với chuyển động montage

**Thách thức:** Nếu chuyển socket ngay khi montage bắt đầu hoặc kết thúc, vũ khí sẽ nhảy vị trí trước hoặc sau thời điểm tay nhân vật thực sự chạm vào nó.

**Giải pháp hiện tại:** Anim Notify riêng theo weapon được đặt tại contact frame. Boolean `AttachToHand` chọn socket lưu trữ hoặc socket tay, sau đó gọi hàm dùng chung `AttachToCharacter`.

### Chuyển giữa physics state và attached state

**Thách thức:** Vũ khí trong world cần collision/physics, còn equipped weapon phải đi theo character socket mà không xung đột với physics.

**Giải pháp hiện tại:** Equip path tắt collision, attach vào socket cấu hình, rồi tắt simulation; drop interface khôi phục world behavior trên actor mới được spawn.

### Hỗ trợ nhiều bộ animation vũ khí

**Thách thức:** Mỗi combat type cần idle/movement pose phù hợp trong khi vẫn dùng chung movement logic.


## 5. Asset Pipeline


1. Import hoặc migrate weapon mesh, material và texture vào project. Source weapon content xuất hiện trong thư mục `InfinityBladeWeapons`.
2. Kiểm tra scale, forward axis, pivot, material assignment, collision và physics asset nếu dùng Skeletal Mesh.
3. Tạo weapon Blueprint kế thừa `BP_Weapon_Base` / `BP_BaseItem`, sau đó gán mesh component phù hợp.
4. Cấu hình `WeaponType`, `CombatType`, `AttachToSocket`, `HandSocket` cho weapon variant.
5. Thêm category vào `E_CombatType` nếu chưa tồn tại.
6. Tạo/import idle, locomotion và swap animation cho category đó, retar.
7. Đặt attach notify tương ứng vào từng swap montage tại đúng frame tay chạm vũ khí hoặc vũ khí chạm vị trí cất, rồi cấu hình `AttachToHand`.
8. Thêm animation asset vào graph lựa chọn của cả `Idle` và `Walk / Run`, đồng thời đặt blend time phù hợp.
9. Đặt vũ khí trong test map và kiểm tra trace, pickup, swap, chuyển socket, drop, collision, physics và đồng bộ animation.
10. Tải animation cần thiết từ Mixamo dưới dạng FBX. Có thể import một source character kèm skin để tạo Mixamo skeleton
