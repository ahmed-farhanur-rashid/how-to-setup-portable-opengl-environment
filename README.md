# Portable OpenGL Environment (Windows x86_64)

This guide explains how to create a self-contained, portable OpenGL development environment for 64-bit Windows (x86_64). The environment bundles:

- GLAD (OpenGL loader)
- GLFW (window + input)
- FreeGLUT (optional compatibility library)
- A prebuilt MinGW-w64 toolchain so the folder can be moved between machines

After following these steps you will be able to build projects using the included `Makefile` and `build.bat` without installing system-wide dependencies.

## Folder layout

Expected layout (top-level of the portable folder):

```
OpenGL_Portable
├─ build/
├─ include/
│     ├─ GL/
│     ├─ glad/
│     ├─ GLFW/
│     └─ KHR/
├─ lib/
│     └─ x64/
├─ mingw64/
├─ src/
│     └─ main.cpp
├─ build.bat
├─ freeglut.dll
├─ glfw3.dll
└─ Makefile
```

## Prerequisites

- A Windows machine with internet access to download the libraries.
- A ZIP extractor (Windows Explorer, 7-Zip, etc.).

## Step-by-step setup

All example commands below assume you're running from the root of your portable folder. Replace `C:\path\to\download` with the actual download/extract path you used.

1) Create the directory layout

```powershell
mkdir build
mkdir include\GL
mkdir include\glad
mkdir include\GLFW
mkdir include\KHR
mkdir lib\x64
mkdir mingw64
mkdir src
```

2) Download and install GLAD

- Go to https://glad.dav1d.de/
- Select:
  - Language: C/C++
  - API: gl
  - Profile: core
  - Version: 4.6
  - Check **Generate a loader** and the **local files** option
- Download the generated ZIP and extract it.

Copy the files from the extracted glad package:

```powershell
# adjust the source path as needed
Copy-Item 'C:\path\to\glad\src\glad.c' -Destination src\
Copy-Item 'C:\path\to\glad\include\glad\glad.h' -Destination include\glad -Recurse
Copy-Item 'C:\path\to\glad\include\KHR\khrplatform.h' -Destination include\KHR -Recurse
```

3) Download and install GLFW (64-bit Windows binaries)

- Download the **64-Bit Windows binaries** from https://www.glfw.org/download.html and extract.
- From the extracted archive:
  - Copy the `include\GLFW` folder to the portable environment root as `./GLFW` (this matches the requested structure)
  - Copy the `lib-mingw-w64\bin\glfw3.dll` to the environment root (so it sits next to `build\main.exe` at runtime)
  - Copy the import library `lib-mingw-w64\lib\libglfw3.a` to `lib\x64`

Example commands:

```powershell
Copy-Item 'C:\path\to\glfw\include\GLFW' -Destination .\GLFW -Recurse
Copy-Item 'C:\path\to\glfw\lib-mingw-w64\bin\glfw3.dll' -Destination .\
Copy-Item 'C:\path\to\glfw\lib-mingw-w64\lib\libglfw3.a' -Destination lib\x64\
```

4) Download and install FreeGLUT

- Download FreeGLUT (development zip) from https://www.transmissionzero.co.uk/software/freeglut-devel/ and extract.
- Copy the contents:

```powershell
Copy-Item 'C:\path\to\freeglut\lib\*' -Destination lib\x64 -Recurse
Copy-Item 'C:\path\to\freeglut\include\*' -Destination include -Recurse
Copy-Item 'C:\path\to\freeglut\bin\freeglut.dll' -Destination .\
```

Note: pick the MinGW-built import libraries (e.g. `.a`) if the archive includes multiple variants.

5) Download and install prebuilt MinGW-w64 (winlibs)

- Visit https://winlibs.com/ and choose the latest **Win64 (Without LLVM/CLANG)** release.
- Extract the archive so the MinGW toolchain sits in `mingw64` at the repo root.

Example (adjust archive name and paths):

```powershell
Expand-Archive -Path 'C:\path\to\winlibs-x86_64-posix-seh.zip' -DestinationPath .\
# If extraction creates a folder like 'winlibs-x86_64-posix-seh-...':
Move-Item 'winlibs-x86_64-posix-seh-*' mingw64
```

6) Ensure sources are present

- Put your application sources in `src\`, at minimum `src\main.cpp` and `src\glad.c` (glad.c comes from step 2).

7) Makefile and `build.bat`

Create the `Makefile` and `build.bat` in the project root (contents below). The repository example uses `lib\x64` for 64-bit import libraries and places `glfw3.dll`/`freeglut.dll` in the project root so the built executable can find them at runtime.

`Makefile`:

```makefile
CC       = g++
CFLAGS   =
INCLUDES = -Iinclude
LIBS     = -Llib/x64 -Llib \
           -lglfw3 -lfreeglut \
           -lopengl32 -lglu32 -lgdi32 -lwinmm -luser32 \
           -static-libgcc -static-libstdc++
SOURCES  = src/main.cpp src/glad.c


all: build/main.exe
build/main.exe: $(SOURCES)
	$(CC) $(INCLUDES) $(CFLAGS) $(SOURCES) $(LIBS) -o build/main.exe
clean:
	-rm -f build/main.exe
```

`build.bat`:

```bat
@echo off
setlocal

cd /d "%~dp0"
set PATH=%~dp0mingw64\bin;%PATH%
echo Building...
mingw32-make -f Makefile
if errorlevel 1 (
    echo.
    echo Build FAILED. See errors above.
    pause
    exit /b 1
)
echo.
echo Build succeeded! Running build\main.exe ...
echo.
build\main.exe
```

8) Build and run

- Run `build.bat` from the repository root (double-click or run from cmd/powershell):

```powershell
.\build.bat
```

The script temporarily prepends `mingw64\bin` to `PATH`, invokes `mingw32-make`, and then runs the produced `build\main.exe`.

## Portability and distribution

- The portable bundle includes the toolchain and runtime DLLs, so you can zip the whole folder and move it to another Windows x86_64 machine.
- `build.bat` uses `%~dp0` so it works from any folder without editing.

## Troubleshooting

- Linker errors (undefined references):
  - Ensure the import libraries in `lib\x64` match your toolchain (x64 MinGW import libs). Mixing x86 and x64 will fail.
  - Confirm the `-Llib/x64 -Llib` paths are correct and the `.a` import libraries exist.
- Missing headers: ensure `include\glad\glad.h`, `include\KHR\khrplatform.h`, and `GLFW` headers are in the `include` or `GLFW` locations you expect.
- Missing DLL at runtime: if `build\main.exe` complains about a missing `*.dll`, copy that DLL to the same folder as the executable or ensure the DLL is on `PATH`.
- If `mingw32-make` is not found: ensure `mingw64\bin` is present and the batch script sets `%PATH%` correctly (it does).

## Minimal example (main.cpp)

You can start with a very small program that creates a window with GLFW and loads GL with GLAD. Put this in `src\main.cpp` and then run `build.bat`.

```cpp
#include <glad/glad.h>
#include <GLFW/glfw3.h>
#include <iostream>

int main() {
    if (!glfwInit()) return -1;
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 4);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 6);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);

    GLFWwindow* win = glfwCreateWindow(640, 480, "GLFW + GLAD", NULL, NULL);
    if (!win) { glfwTerminate(); return -1; }
    glfwMakeContextCurrent(win);

    if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress)) {
        std::cerr << "Failed to initialize GLAD\n";
        return -1;
    }

    std::cout << "OpenGL " << glGetString(GL_VERSION) << "\n";

    while (!glfwWindowShouldClose(win)) {
        glfwSwapBuffers(win);
        glfwPollEvents();
    }
    glfwDestroyWindow(win);
    glfwTerminate();
    return 0;
}
```

## Final notes

- This setup intentionally keeps everything local to the repository root so the folder can be moved or zipped and used on other compatible Windows machines.
- If you want to reduce size for distribution, remove unused MinGW toolchain pieces after building, but keep the runtime DLLs (`glfw3.dll`, `freeglut.dll`) and any required redistributables.

---
