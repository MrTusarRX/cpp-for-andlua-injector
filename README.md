# Memory Injection & Modification for Games  

## Overview  
This project demonstrates how to inject and modify a game's memory at runtime on Android. It uses C++ to locate the game's process, find the base address of a specific library (`libil2cpp.so`), and modify memory values using `pwrite64`.  

## Features  
- **Process Scanning** – Finds the target game’s process ID (`PID`).  
- **Memory Editing** – Modifies specific memory addresses in the game's runtime.  
- **Library Base Address Detection** – Retrieves the base address of `libil2cpp.so`.  
- **Float & DWORD Modification** – Writes new values to memory dynamically.  

## How It Works  
1. **Find the Game Process**  
   - The script scans `/proc` for the target game package (`com.dts.freefireth`).  
2. **Open the Game’s Memory**  
   - It opens `/proc/<PID>/mem` with read-write access.  
3. **Locate the Library Base Address**  
   - Searches for `libil2cpp.so` and retrieves its base address.  
4. **Modify Memory Values**  
   - Writes a float value (`2.0f`) to a specific offset (`0x4104C1C`).  

## Code Example  
```cpp
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <dirent.h>
#include <stdlib.h>

int handle;
long int get_module_base(int pid, const char *module_name) {
    FILE *fp;
    long addr = 0;
    char filename[32], line[1024], *pch;
    snprintf(filename, sizeof(filename), "/proc/%d/maps", pid);
    fp = fopen(filename, "r");
    if (fp) {
        while (fgets(line, sizeof(line), fp)) {
            if (strstr(line, module_name)) {
                pch = strtok(line, "-");
                addr = strtoul(pch, NULL, 16);
                break;
            }
        }
        fclose(fp);
    }
    return addr;
}

int WriteBaseAddress_FLOAT(long int addr, float value) {
    pwrite64(handle, &value, 4, addr);
    return 0;
}

int getPID(const char *packageName) {
    DIR *dir = opendir("/proc");
    struct dirent *ptr;
    char filepath[256], filetext[128];
    FILE *fp;
    if (dir) {
        while ((ptr = readdir(dir))) {
            if (ptr->d_type != DT_DIR) continue;
            sprintf(filepath, "/proc/%s/cmdline", ptr->d_name);
            fp = fopen(filepath, "r");
            if (fp) {
                fgets(filetext, sizeof(filetext), fp);
                if (strcmp(filetext, packageName) == 0) {
                    fclose(fp);
                    closedir(dir);
                    return atoi(ptr->d_name);
                }
                fclose(fp);
            }
        }
        closedir(dir);
    }
    return 0;
}

int main() {
    int pid = getPID("com.dts.freefireth");  // Target Game Package
    if (pid == 0) {
        puts("Game not found!");
        return 1;
    }

    char memPath[64];
    sprintf(memPath, "/proc/%d/mem", pid);
    handle = open(memPath, O_RDWR);
    if (handle == -1) {
        puts("Failed to open memory!");
        return 1;
    }

    long int libBase = get_module_base(pid, "libil2cpp.so");
    WriteBaseAddress_FLOAT(libBase + 0x4104C1C, 2.0f);  // Modify Value

    close(handle);
    return 0;
}
