Challenge: Compile the greeter library, generate interop bindings for Kotlin/Native, and invoke the library function. 

## Step 0: Compile kgreeter
0. Download CLion: [https://www.jetbrains.com/clion/download](https://www.jetbrains.com/clion/download)
1. clone the project repo: `git clone https://github.com/mutexkid/kotlin-native-workshop`
2. open a terminal and go to the root directory of the repo. type `./gradlew tasks` to list the available tasks for the project. Notice the following output: 
	
	```text
	Build tasks
	-----------
	assemble - Assembles the outputs of this project.
	build - Assembles and tests this project.
	clean - Deletes the build directory.
	compileKonan - Compiles all the Kotlin/Native artifacts
	compileKonanKgreeter - Build the Kotlin/Native executable 'compileKonanKgreeter' for all supported and declared targets
	compileKonanKgreeterMacbook - Build the Kotlin/Native executable 'compileKonanKgreeterMacbook' for target 'macbook'
	compileKonanKgreeterMacos_x64 - Build the Kotlin/Native executable 'compileKonanKgreeterMacos_x64' for target 'macos_x64'
	
	```
	
4. run the task `compileKonanKgreeterMacbook` by entering `./gradlew compileKonanKgreeterMacbook` to create a native binary for macbook. Note that on the first time running, the kotlin/native compiler and cinterop tools will be downloaded - wait patiently!
5. Notice that the `build` directory has been created. Run the native binary that was compiled by entering:  `./build/konan/bin/macos_x64/kgreeter.kexe`.
6.  Observe the following output: 
`Hello from Kotlin/Native!`


## Step 1: Compile the cgreeter c library

2. Observe the `CmakeLists.txt` file in the directory `./scr/libs/cgreeter`. This is a configuration tool similar to `gradle` for c projects. To build the c library, you first must generate a Makefile using `cmake` (command line tool). Change to the `./src/libs/cgreeter` directory and type `cmake CmakeLists.txt` to generate the build config. 
3. Compile the library by running `make` within the `./src/libs/cgreeter` directory. 
4. Notice that a static library was generated by the compiler: `./src/libs/cgreeter/libcgreeter.a`


## Step 2: Generate interop stubs from kgreeter
1. Now that the library is compiled, you will generate interop bindings that allow you to invoke the c function from kotlin/native. The `cinterop` tool (part of the kotlin/native toolchain) requires 2 things: a header file (similar to a java interface), and a library (either a .a or .dylib on *nix machines). 
2. Open cgreeter.c. Notice the signature of the `sayHelloInC` function. 
2. Define a header file for the cgreeter library so that the interop tool can create a matching stub file. Create a file called `cgreeter.h` and add the signature _only_ for sayHelloInC: 

	`cgreeter.h:`
	
	```c
	void sayHelloInC(char *name);
	
	```

3. Next, you will generate the interop bindings that will allow you to call the function from kotlin/native. Kotlin/Native uses the `cinterop` tool to allow you to invoke functions in the c library you compiled from Kotlin, and the `cinterop` tool takes the specification for the library you wish to bind to as a gradle defintion. 
4. Add an interop entry to the `build.gradle` file, and the cgreeter artifact dependency to the kgreeter program definition. After completing, your build.gradle should look like the following: 

	`build.gradle:`
	
	```
	plugins {
	    id "org.jetbrains.kotlin.konan" version "0.8"
	}
	
	konanArtifacts {
	           
	    interop('cgreeter', targets: ['macbook']) {
	       defFile 'cgreeter.def'
	    }
	    program('kgreeter', targets: ['macbook']) {
	        srcDir "${project.rootDir}/src/main/kotlin"
	        libraries {
	            artifact 'cgreeter'
	        }
	    }
	}
	
	```

3. Notice that `cgreeter.def` is referenced in the build.gradle but doesn't exist yet. Time to fix that. Create a file called `cgreeter.def` in the project root, and add the following: 

	
	`cgreeter.def:`
	
	```text
	package = cgreeter
	headers = cgreeter.h
	compilerOpts=-I./src/libs/cgreeter/
	libraryPaths =./src/libs/cgreeter/
	staticLibraries = libcgreeter.a 
	```
	
	The definition specifies the location of the header and binary for the cgreeter library so that the cinterop tool can generate bindings.

4. Verify the cinterop config works correctly by running `./gradlew compileKonanKgreeterMacbook`. Notice that a new step was added to the gradle output: `> :compileKonanCgreeterMacos_x64`. In this step, the interop bindings were generated for the c library and added to the project (under `build/konan/bin/libs/macos_x64/cgreeter.klib-build` )

## Step 3: Call the interopped c library

1. The interop stubs have been created for the c libary. Now you can call the library from Kotlin/Native. Add the following to hello.kt: 

	`hello.kt`
	
	```kotlin
	import kotlinx.cinterop.cValuesOf
	import kotlinx.cinterop.cstr
	
	fun main(args: Array<String>) {
	    println("Hello from Kotlin/Native!")
	    cgreeter.sayHelloInC("Josh".cstr)
	}
	```
2. Compile and run the program: `./gradlew compileKonanKgreeter && ./build/konan/bin/macos_x64/kgreeter.kexe`

3. Notice several things about the file: 
	- `import kotlinx.cinterop` : K/N cinterop features are imported from this package and include types to represent C Pointers, C Variables, and many other useful features: [https://kotlinlang.org/docs/tutorials/native/interop-with-c.html](https://kotlinlang.org/docs/tutorials/native/interop-with-c.html)
	- `cgreeter.sayHelloInC()` : here you imported the cgreeter library that you compiled and generated bindings for
	- `"Josh".cstr` You converted a Kotlin string type to the corresponding c type, a `char *` . (using the kotlinx.cinterop `.cstr` extension)


