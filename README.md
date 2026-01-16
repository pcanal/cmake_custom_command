As first reported at https://github.com/root-project/root/issues/20907
and reproduced standalone with this repository,
there are circumstances when the use of `add_custom_command` to
generate a source and an auxiliary file and
specifying some `IMPLICIT_DEPENDS` where
you can get spurious rebuild when using `Unix Makefiles`.

To reproduce the problem:
```
mkdir build
cd build/
cmake .. -G 'Unix Makefiles'
# (Step 1) Fully build project, now call cmake
make
# (Step 2) Reconfigure to reset the custom command generated dependencies
cmake ..
# Make a change
touch ../sub1/nestedheader.h
# (Step 3) This should rebuild the project fully
make
# (Step 4) This should be doing nothing but does 
make
```
The mechanism of the failures is  


On step (3), because of step (2), `sub1/CMakeFiles/main.dir/depend.make ` is empty
and thus `gensource.cxx` depends only on the explicit dependencies until that file is regenerated.

Inside step 3, one of the sub step (3.1) is:
```
- /usr/sbin/gmake -s -f sub1/CMakeFiles/main.dir/build.make sub1/CMakeFiles/main.dir/depend
- - Checks both genhelper.txt and gensource.cxx 
- - - No need to remake target 'sub1/gensource.cxx'
- - - Prerequisite 'sub1/gensource.cxx' is older than target 'sub1/genhelper.txt'
```
and generates the full content of sub1/CMakeFiles/main.dir/depend.make.
This steps 3.1 is also the step usually responsible for updating the timestamp of genhelper.txt to be newer than
gensource.cxx.  This is not needed/executed in this iteration.

Then inside 3, the next sub step (3.2) is:
```
- run /usr/sbin/gmake -s -f sub1/CMakeFiles/main.dir/build.make sub1/CMakeFiles/main.dir/build
- -    Must remake target 'sub1/gensource.cxx'.
- - sub1/CMakeFiles/main.dir/build.make: update target 'sub1/gensource.cxx' due to: sub1/nestedheader.h
- *BUT* genhelper.txt is not checked (it is not a dependency of `build`).
```
which loads the generated dependency file and thus notice that `gensource.cxx` needs to be
regenerated.  The generation produces `genhelper.txt` first and the `gensource.cxx`.  So 
after this step `genhelper.txt` is older than `gensource.cxx`.  The updating of the timestamp of `genhelper.txt`
is not run (it is not part of the dependency tree of sub1/CMakeFiles/main.dir/build).

Now on step 4, we run the same sub-step (4.1):
```
- run /usr/sbin/gmake -s -f core/CMakeFiles/G__Core.dir/build.make core/CMakeFiles/G__Core.dir/depend
```
this time `make` notices that:
- `gensource.cxx` does not need updating
- `genhelper.txt` needs to have its time stamp updated.
- so `genhelper.txt`'s timestamp is updated even-though it has not changed

Subsequently anything that depends on `genhelper.txt` is needlessly rebuild.


And the output is:

```
# mkdir build
# cd build/
# cmake ..
-- The C compiler identification is GNU 16.0.0
-- The CXX compiler identification is GNU 16.0.0
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: /usr/lib64/ccache/cc - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/lib64/ccache/c++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done (0.4s)
-- Generating done (0.0s)
-- Build files have been written to: /github/home/ROOT-CI/cmake_report/build
```
```
# make
[ 16%] Generating gensource.cxx, genhelper.txt
2026-01-16 23:09:22.491557133 +0000
2026-01-16 23:09:24.495623105 +0000
[ 33%] Building CXX object sub1/CMakeFiles/main.dir/gensource.cxx.o
[ 50%] Linking CXX shared library libmain.so
[ 50%] Built target main
[ 66%] Generating othergensource.cxx
[ 83%] Building CXX object sub2/CMakeFiles/sub.dir/othergensource.cxx.o
[100%] Linking CXX shared library libsub.so
[100%] Built target sub
```
```
# cmake .
-- Configuring done (0.0s)
-- Generating done (0.0s)
-- Build files have been written to: /github/home/ROOT-CI/cmake_report/build
(ROOT-CI) [root@84d1e11b589e build]# touch ../sub1/nestedheader.h 
(ROOT-CI) [root@84d1e11b589e build]# make
[ 16%] Generating gensource.cxx, genhelper.txt
2026-01-16 23:09:34.762961098 +0000
2026-01-16 23:09:36.767027071 +0000
[ 33%] Building CXX object sub1/CMakeFiles/main.dir/gensource.cxx.o
[ 50%] Linking CXX shared library libmain.so
[ 50%] Built target main
[ 66%] Generating othergensource.cxx
[ 83%] Building CXX object sub2/CMakeFiles/sub.dir/othergensource.cxx.o
[100%] Linking CXX shared library libsub.so
[100%] Built target sub
```
```
# make
[ 50%] Built target main
[ 66%] Generating othergensource.cxx
[ 83%] Building CXX object sub2/CMakeFiles/sub.dir/othergensource.cxx.o
[100%] Linking CXX shared library libsub.so
[100%] Built target sub
```

