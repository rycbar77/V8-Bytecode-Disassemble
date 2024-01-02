# V8-Bytecode-Disassemble
Disassemble V8 Ignition bytecode.

Test on linux.
## 0.Credit
Slightly modified from [v8dasm](https://github.com/noelex/v8dasm).

## 1.Modify V8 Source Code
In `code-serializer.cc`, insert the following lines after the serialization is completed (after maybe_result is successfully converted to result) in function `CodeSerializer::Deserialize`:

```c++
result->GetBytecodeArray(isolate).Disassemble(std::cout);
std::cout << std::flush;
```

Modify function `SharedFunctionInfo::SharedFunctionInfoPrint` in `objects-printer.cc`:

```diff
diff --git a/src/diagnostics/objects-printer.cc b/src/diagnostics/objects-printer.cc
index 2ec89d58962..7665668d8ae 100644
--- a/src/diagnostics/objects-printer.cc
+++ b/src/diagnostics/objects-printer.cc
@@ -1652,6 +1652,14 @@ void SharedFunctionInfo::SharedFunctionInfoPrint(std::ostream& os) {
     os << "<none>";
   }
   os << "\n";
+
+  os << "\n; --- start SharedFunctionInfoDisassembly\n";
+  Isolate* isolate = GetIsolate();
+  if (this->HasBytecodeArray()) {
+    this->GetBytecodeArray(isolate).Disassemble(os);
+    os << std::flush;
+  }
+  os << "; --- end SharedFunctionInfoDisassembly\n";
 }
```

Modify function HeapObject::HeapObjectShortPrint in objects.cc.

```diff
diff --git a/src/objects/objects.cc b/src/objects/objects.cc
index 4616ef7ab74..9b1e67a2f4a 100644
--- a/src/objects/objects.cc
+++ b/src/objects/objects.cc
@@ -1860,6 +1860,7 @@ void HeapObject::HeapObjectShortPrint(std::ostream& os) {
     os << accumulator.ToCString().get();
     return;
   }
+  int len;
   switch (map(cage_base).instance_type()) {
     case MAP_TYPE: {
       os << "<Map";
@@ -1941,11 +1942,27 @@ void HeapObject::HeapObjectShortPrint(std::ostream& os) {
          << "]>";
       break;
     case FIXED_ARRAY_TYPE:
-      os << "<FixedArray[" << FixedArray::cast(*this).length() << "]>";
+      len = FixedArray::cast(*this).length();
+      os << "<FixedArray[" << len << "]>";
+
+      if (len) {
+        os << "\n; #region FixedArray\n";
+        FixedArray::cast(*this).FixedArrayPrint(os);
+        os << "; #endregion";
+      }
+
       break;
     case OBJECT_BOILERPLATE_DESCRIPTION_TYPE:
-      os << "<ObjectBoilerplateDescription[" << FixedArray::cast(*this).length()
-         << "]>";
+      len = FixedArray::cast(*this).length();
+      os << "<ObjectBoilerplateDescription[" << len << "]>";
+
+      if (len) {
+        os << "\n; #region ObjectBoilerplateDescription\n";
+        ObjectBoilerplateDescription::cast(*this)
+            .ObjectBoilerplateDescriptionPrint(os);
+        os << "; #endregion";
+      }
+
       break;
     case FIXED_DOUBLE_ARRAY_TYPE:
```

## Configure and Build V8
Make sure to check if the bytecode you have works with `Pointer Compression` or not.

For example, it will cause different length-of-steps when executing `CopySlots` fuction.

```c++
  void CopySlots(Address* dest, int number_of_slots) {
    base::AtomicWord* start = reinterpret_cast<base::AtomicWord*>(dest);
    base::AtomicWord* end = start + number_of_slots;
    for (base::AtomicWord* p = start; p < end;
         ++p, position_ += sizeof(base::AtomicWord)) {
      base::AtomicWord val;
      memcpy(&val, data_ + position_, sizeof(base::AtomicWord));
      base::Relaxed_Store(p, val);
    }
  }

#ifdef V8_COMPRESS_POINTERS
  void CopySlots(Tagged_t* dest, int number_of_slots) {
    AtomicTagged_t* start = reinterpret_cast<AtomicTagged_t*>(dest);
    AtomicTagged_t* end = start + number_of_slots;
    for (AtomicTagged_t* p = start; p < end;
         ++p, position_ += sizeof(AtomicTagged_t)) {
      AtomicTagged_t val;
      memcpy(&val, data_ + position_, sizeof(AtomicTagged_t));
      base::Relaxed_Store(p, val);
    }
  }
#endif
```

Configure with the following arguments if `Pointer Compression` is disabled.

```
is_debug = false
target_cpu = "x64"
v8_enable_backtrace = true
v8_enable_slow_dchecks = true
v8_optimized_debug = false
is_component_build = false
v8_static_library = true
v8_enable_disassembler = true
v8_enable_object_print = true
use_custom_libcxx = false
use_custom_libcxx_for_host = false
v8_use_external_startup_data = false
enable_iterator_debugging = false
use_goma=false
goma_dir="None"
v8_enable_pointer_compression = false
```

## Custom Code Cache loader
Write a program to call `ScriptCompiler::CompileUnboundScript` to load code cache from file. `v8dasm.cpp` is a sample program for linux.

Build

```bash
clang++ v8dasm.cpp -std=c++17 -I../../include -Lobj -lv8_libbase -lv8_libplatform -lv8_base_without_compiler -lwee8 -o v8dasm
```
