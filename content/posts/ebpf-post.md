+++
date = '2025-12-09T11:39:13+08:00'
draft = false
title = 'eBPF 開發方式演進比較'
+++

本文件比較了 **傳統方式 (早期 API / bpftool)** 與 **現代方式 (libbpf Skeleton)** 在 eBPF 程式開發中的差異。

本報告整理了 eBPF 程式從「早期手動載入」到「libbpf skeleton」的演進歷程，並對比各階段在 **Program Load** 與 **Map Create** 的差異。

---

## 🚩 eBPF 開發四個階段

1. **Raw syscall 方式**

   - 完全透過 `bpf()` 系統呼叫 (`bpf_create_map`, `bpf_prog_load`) 操作。
   - 開發者需自己維護 map 的參數、program 載入與 attach。
   - 最繁瑣但最底層，主要用於研究或特殊需求。

2. **bpftool CLI 方式**

   - 使用 `bpftool prog load`、`bpftool map create` 等指令操作。
   - 適合測試與手動操作，不需要寫太多 userspace code。
   - 不適合大型專案或動態管理。

3. **libbpf Legacy API**

   - 利用 libbpf 提供的函式：  
     `bpf_object__open_file()` + `bpf_object__load()`
   - 可以自動解析 ELF 檔並建立 map，比 raw syscall 簡化許多。
   - 但程式碼仍較冗長，需要手動 attach program。

4. **libbpf Skeleton (Modern API)**
   - 使用 `bpftool gen skeleton prog.o > prog.skel.h` 生成 skeleton header。
   - 在 userspace 呼叫 `xxx_bpf__open/load/attach()` 即可自動完成 **程式載入 + map 建立 + attach**。
   - 目前主流開發方式。

---

## 📊 功能比較表

| 功能面向                 | Raw syscall 方式                                                                 | bpftool CLI 方式                                          | libbpf Legacy API                                | libbpf Skeleton (Modern API)                                |
| ------------------------ | -------------------------------------------------------------------------------- | --------------------------------------------------------- | ------------------------------------------------ | ----------------------------------------------------------- |
| **Program 編譯**         | `clang -target bpf -c prog.bpf.c -o prog.o`                                      | 相同                                                      | 相同                                             | 相同 + `bpftool gen skeleton`                               |
| **Program Load**         | `bpf_prog_load()` / `bpf_prog_load_xattr()`                                      | `bpftool prog load prog.o /sys/fs/bpf/my_prog`            | `bpf_object__open_file()` + `bpf_object__load()` | `skel = xxx_bpf__open(); xxx_bpf__load(skel);`              |
| **Attach Program**       | `bpf_prog_attach()`                                                              | `bpftool prog attach prog /cgroup /sys/fs/cgroup/unified` | 需手動呼叫 `bpf_program__attach_*()`             | `xxx_bpf__attach(skel)`                                     |
| **Map 建立**             | `bpf_create_map(BPF_MAP_TYPE_HASH, …)`                                           | `bpftool map create /sys/fs/bpf/my_map type hash …`       | ELF 內 map 會由 libbpf 建立                      | `.bpf.c` 用 `SEC(".maps")` 定義，libbpf 自動完成            |
| **Map 使用 (userspace)** | 需自己用 `bpf_obj_get("/sys/fs/bpf/my_map")` 取得 fd，再 `bpf_map_update_elem()` | 只能透過 bpftool 指令存取，或再搭配 `bpf_obj_get()`       | 透過 `bpf_map__fd()` 取得 fd                     | skeleton 自動生成成員：`skel->maps.my_hash`                 |
| **Map 支援類型**         | HASH、ARRAY、PERF_EVENT_ARRAY、RINGBUF… 都需手動建立                             | 同左，必須 `bpftool map create`                           | ELF 定義自動建立                                 | `.bpf.c` 宣告 `__uint(type, …)`，自動支援 HASH/RINGBUF/PERF |

## 📌 Map 建立方式對照

### 🔹 Raw syscall 方式

```c
int hash_fd = bpf_create_map(BPF_MAP_TYPE_HASH,
                             sizeof(int),     // key size
                             sizeof(long),    // value size
                             1024,            // max entries
                             0);              // flags

int ring_fd = bpf_create_map(BPF_MAP_TYPE_RINGBUF,
                             1 << 24,         // 16MB buffer
                             0, 0, 0);

```

### 🔹 bpftool CLI 方式

```bash
# 建立 HASH map
bpftool map create /sys/fs/bpf/my_hash \
    type hash key 4 value 8 entries 1024

# 建立 RINGBUF map
bpftool map create /sys/fs/bpf/my_ring \
    type ringbuf entries 16384
```

### 🔹 libbpf Skeleton 方式

在 `.bpf.c` 中定義即可：

```c
// Hash map
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __type(key, int);
    __type(value, long);
    __uint(max_entries, 1024);
} my_hash SEC(".maps");

// Ring buffer
struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 1 << 24);
} my_ring SEC(".maps");
```

Userspace 使用：

```
bpf_map_update_elem(bpf_map__fd(skel->maps.my_hash), &pid, &val, BPF_ANY);
```

## 📌 差異總結

- **Raw syscall**：最底層，繁瑣但彈性最高。
- **bpftool CLI**：適合快速測試與 debug，不適合大型專案。
- **libbpf Legacy**：已有部分自動化，但仍需手動 attach。
- **libbpf Skeleton**：現代主流方式，自動完成 **map 建立 + 程式載入 + attach**，開發效率最高。

---

---

## 名詞說明

- **傳統方式 (Early API / bpftool)**  
  透過 `bpftool prog load` 或 `bpf_prog_load_xattr()`，以及 `bpf_create_map_xattr()` 等 API，在使用者空間手動建立 Map 與載入程式。

- **現代方式 (libbpf Skeleton)**  
  使用 `bpftool gen skeleton` 產生的 Skeleton C code，透過 `execmon_bpf__open()` 與 `execmon_bpf__load()` 由 libbpf 自動處理程式載入與 Map 建立。

---

## 差異比較表

| 項目               | 傳統方式 (Early API / bpftool)                                                              | 現代方式 (libbpf Skeleton)                                           |
| ------------------ | ------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| **Map 建立**       | 需自行呼叫 `bpf_create_map()` 或 `bpf_create_map_xattr()` 指定 key/value 大小、最大 entries | Skeleton 根據 `.bpf.c` 自動生成 Map，透過 `execmon_bpf__open()` 取得 |
| **Map 類型支援**   | `BPF_MAP_TYPE_HASH`、`BPF_MAP_TYPE_RINGBUF`、`BPF_MAP_TYPE_PERF_EVENT_ARRAY` 等需手動建立   | Map 類型在 BPF 程式內定義，Skeleton 自動建立並回傳 FD                |
| **程式載入**       | `bpftool prog load <obj>` 或 `bpf_prog_load_xattr()` 手動載入 ELF                           | `execmon_bpf__load()` 由 libbpf 處理程式載入與驗證                   |
| **Map 綁定程式**   | 手動執行 `bpftool map update` 或 `bpf_map_update_elem()`                                    | Skeleton 會自動完成 Map 與程式綁定                                   |
| **使用者空間程式** | 需自行維護所有 Map FD 與程式 FD                                                             | Skeleton 提供結構化 API，使用更簡單                                  |
| **開發維護成本**   | 高，需要熟悉所有 eBPF syscall 與 bpftool 指令                                               | 低，libbpf 自動處理細節                                              |

---

## 總結

- **傳統方式**：靈活，但需大量手動操作，適合學習與低階除錯。
- **現代方式**：透過 libbpf Skeleton，大幅簡化開發，適合實際專案使用。
