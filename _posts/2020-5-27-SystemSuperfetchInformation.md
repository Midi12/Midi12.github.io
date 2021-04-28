---
layout: post
title: NtQuerySystemInformation SystemSuperfetchInformation update
orig_link: https://www.unknowncheats.me/forum/general-programming-and-reversing/397104-ntquerysysteminformation-systemsuperfetchinformation.html
---

How investigating a `NtQuerySystemInformation` defect behavior leads to finding a feature update.

---

*All content below is for educational purpose only*

<a href="{{ page.orig_link }}" target="_blank">Original link</a>

---

While trying to play with physical memory I found [@Waryas](https://www.unknowncheats.me/forum/members/129978.html) nice [thread](https://www.unknowncheats.me/forum/anti-cheat-bypass/240271-umpmlib-read-memory-process-usermode-2-dkom-trick.html) and tried his `NtQuerySystemInformation` `SystemSuperfetchInformation` code.

Unfortunately, the code didn't work as is, so I investigated.
Calling `NtQuerySystemInformation` with `SystemSuperfetchInformation` class internally calls `nt!PfQuerySuperfetchInformation` which calls in turn `nt!PfpMemoryRangesQuery`.

In Waryas's code the struct receiving class information is like below

```cpp
typedef struct _PF_MEMORY_RANGE_INFO {
    ULONG Version;
    ULONG RangeCount;
    PF_PHYSICAL_MEMORY_RANGE Ranges[ANYSIZE_ARRAY];
} PF_MEMORY_RANGE_INFO, * PPF_MEMORY_RANGE_INFO;
```

and Version field has to be initialized with `0x1` value.

After tracing the call with \<insert your favorite kernel debugger there\> you'll end up at `nt!PfpMemoryRangesQuery+0x39`.

```fffff800`7817dd55 833e02           cmp     dword ptr [rsi], 2```

This checks for `Version` field of `_PF_MEMORY_RANGE_INFO` equals `0x2` (stored in `rsi` register) which doesn't add up with Waryas's code.

On my system (Windows version `10.0.18362.1`) it asks for version 2 of the structure.

I don't know which Windows system version 2 of the struct appeared but that's business only if you want/need to support multiple versions of windows. (Thanks to [@JD96](https://www.unknowncheats.me/forum/members/201446.html) it has been added between `10_1709_16299_248` and `10_1803_17134_81`)

As usual with Windows API versioned structure, if the version updates the struct updates.

Let's reverse `nt!PfpMemoryRangesQuery`!

The major difference between versions `1` and `2` of `nt!PfpMemoryRangesQuery` is that it internally uses the new `MmGetPhysicalMemoryRangesEx2` instead of `MmGetPhysicalMemoryRangesEx`.

The only difference between the two functions mentionned above is a new parameter called flags. According to [msdn](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/ntddk/nf-ntddk-mmgetphysicalmemoryrangesex2) the flag is used if the caller wants `FileOnly` ranges. If you only want `FileOnly` ranges set the flags parameter to `1` otherwise set it to `0`.

In the end, only a new member is added to `_PF_MEMORY_RANGE_INFO` structure and it is indeed the flags parameter.

Here is the `_PF_MEMORY_RANGE_INFO_V2` structure :

```cpp
typedef struct __declspec(align(8)) _PF_MEMORY_RANGE_INFO_V2 {
    ULONG version;
    ULONG flags;
    ULONG ranges_count;
    _PF_PHYSICAL_MEMORY_RANGE ranges[ANYSIZE_ARRAY];
};
```

```cpp
static superfetch_config_t& initialize(void) {
    if (!initialize_privileges()) {
        return g_superfetch_config;
    }

    // assign superfecth version dynamically and use the right struct, some system are v1 some are v2 due to windows update
    // (on my system nt!PfpMemoryRangesQuery checks for memory_range_info.Version == 2)

    native::PPF_MEMORY_RANGE_INFO_V2 memory_ranges = nullptr;
    native::SUPERFETCH_INFORMATION sf_info = {};
    std::size_t result_length = 0;
    native::PF_MEMORY_RANGE_INFO_V2 memory_range_info = {};
    memory_range_info.version = 2;

    auto build_info = [=](native::PSUPERFETCH_INFORMATION SuperfetchInfo, PVOID Buffer, ULONG Length, native::SUPERFETCH_INFORMATION_CLASS InfoClass) -> void {
        SuperfetchInfo->Version = SUPERFETCH_VERSION;
        SuperfetchInfo->Magic = SUPERFETCH_MAGIC;
        SuperfetchInfo->Data = Buffer;
        SuperfetchInfo->Length = Length;
        SuperfetchInfo->InfoClass = InfoClass;
    };

    build_info(&sf_info, &memory_range_info, sizeof(memory_range_info), native::SUPERFETCH_INFORMATION_CLASS::SuperfetchMemoryRangesQuery);

    NTSTATUS status = LAZYCALL(NTSTATUS, "ntdll.dll!NtQuerySystemInformation", native::SYSTEM_INFORMATION_CLASS::SystemSuperfetchInformation, &sf_info, sizeof(sf_info), &result_length);
    if (status == STATUS_BUFFER_TOO_SMALL) {
        memory_ranges = static_cast<native::PPF_MEMORY_RANGE_INFO_V2>(HeapAlloc(GetProcessHeap(), HEAP_ZERO_MEMORY, result_length));
        memory_ranges->version = 2;
        build_info(&sf_info, memory_ranges, static_cast<ULONG>(result_length), native::SUPERFETCH_INFORMATION_CLASS::SuperfetchMemoryRangesQuery);
        status = LAZYCALL(NTSTATUS, "ntdll!NtQuerySystemInformation", native::SYSTEM_INFORMATION_CLASS::SystemSuperfetchInformation, &sf_info, sizeof(sf_info), &result_length);
        if (status != STATUS_SUCCESS) {
            return g_superfetch_config;
        }
    } else if (status == STATUS_SUCCESS) {
        memory_ranges = &memory_range_info;
    } else {
        return g_superfetch_config;
    }

    if (memory_ranges->ranges_count == 0x0) {
        return g_superfetch_config;
    }

    native::PPHYSICAL_MEMORY_RUN Node;
    for (ULONG i = 0; i < memory_ranges->ranges_count; i++) {
        Node = reinterpret_cast<native::PPHYSICAL_MEMORY_RUN>(&memory_ranges->ranges[i]);

        g_superfetch_config.physical_memory_ranges.push_back({
            Node->BasePage << PAGE_SHIFT,
            (Node->BasePage + Node->PageCount) << PAGE_SHIFT,
            Node->PageCount,
            ((Node->PageCount << PAGE_SHIFT) >> 10) * 1024		 // kb to byte
        });
    }

    g_superfetch_config.initialized = true;
    return g_superfetch_config;
}
```

Full code can be found [there](https://gist.github.com/Midi12/12823859abc4b18c45587949c65fb38f).