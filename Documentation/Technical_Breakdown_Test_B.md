# Technical Breakdown - Character Animation Tech (TestB)

## 1. Animation Blueprint
- ở Idle và Walk/Run có sử dụng blend space để blend giữa các animation khác nhau tùy theo combat type
- sử dụng State Machine để điều khiển các animation như Jumping, Falling và các animation khác
- sử dụng Layered Blend Per Bone để blend giữa các animation khác nhau
- 
## 2. Blueprint Integration
### 2.1. Weapon Class Hierarchy & Combat Type Enum
- E_CombatType định nghĩa 4 giá trị:  Sword, GreatSword, Axe — dùng làm khóa chuyển đổi animation set và logic equip tương ứng trong cả Character lẫn ABP.
- Cấu trúc kế thừa: BP_BaseItem (gồm 3 component gốc: DefaultSceneRoot, SkeletalBase, StaticBase) → BP_Weapon_Base → các Blueprint cụ thể (BP_Weapon_Sword, BP_Weapon_GreatSword, BP_Weapon_Axe).
- Tách weapon logic ra khỏi Character bằng class hierarchy riêng giúp thêm vũ khí mới chỉ cần tạo Blueprint con kế thừa BP_Weapon_Base, không phải sửa code Character.

### 2.2. Attach Logic (BP_BaseItem — Function AttachToCharacter)
- Function AttachToCharacter có tham số mặc định Attach Socket Name = "Weapon_Socket" và Hand = None — cho phép mọi weapon con gọi cùng một hàm attach mà không cần khai báo lại tên socket riêng lẻ.\

### 2.3. Pickup Flow (BP_ThirdPersonCharacter — Event "Pick Up Weapon")
- Custom event "Pick Up Weapon" trigger khi người chơi nhấn phím E trong lúc overlap với weapon actor
- Event này gọi vào function AttachToCharacter của weapon đang overlap, hoàn tất việc gắn vũ khí vào socket tay nhân vật.

### 2.4. Equip / Switch Weapon Flow (Events "Equip 1 / Equip 2 / Equip 3")
- tách 3 event Equip riêng theo từng phím số giúp dễ debug trực quan trong Event Graph khi test từng loại vũ khí, dù về lâu dài có thể gộp lại để giảm trùng lặp node.

### 2.5. Drop Weapon Flow (Functions DropSword / DropGreatSword / DropAxe)
Mỗi loại vũ khí có function Drop riêng (DropSword, DropGreatSword, DropAxe) thay vì 1 hàm Drop tổng quát — mỗi hàm xử lý detach và reset WeaponStatus/Combat Type phù hợp với loại vũ khí đang cầm tại thời điểm gọi.