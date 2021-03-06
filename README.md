module LibraryMake
------------------

An attempt to simplify building native code for a perl6 module.

This is effectively a small configure script for a Makefile. It will allow you to use the same tools to build your native code that were used to build perl6 itself.

Typically, this will be used in both Build.pm (to support panda installs), and in a standalone Configure.pl script in the src directory (to support standalone testing/building). Note that if you need additional custom configure code, you will currently need to add it to both your Build.pm and to your Configure.pl6

Example Usage
-------------

The below files are examples of what you would write in your own project. The src directory is merely a convention, and the Makefile.in will likely be significantly different in your own project.

/Build.pm

    use Panda::Common;
    use Panda::Builder;
    use LibraryMake;
    use Shell::Command;

    class Build is Panda::Builder {
        method build($workdir) {
            mkpath "$workdir/blib/lib";
            make("$workdir/src", "$workdir/blib/lib");
        }
    }

/src/Configure.pl6

    # Note that this is *not* run during panda install
    # The example here is what the 'make' call in Build.pm does
    use LibraryMake;

    my $destdir = '../lib';
    my %vars = get-vars($destdir);
    process-makefile('.', %vars);

/src/Makefile.in

    all: %DESTDIR%/libfoo%SO%

    %DESTDIR%/libfoo%SO%: libfoo%O%
        %LD% %LDSHARED% %LDFLAGS% %LIBS% %LDUSR%pam %LDOUT%%DESTDIR%/libfoo%SO% libfoo%O%

    libfoo%O%: libfoo.c
        %CC% -c %CCSHARED% %CCFLAGS% %CCOUT%libfoo%O% libfoo.c

/lib/Foo.pm6

    # ...

    use NativeCall;
    use LibraryMake;

    # Find our compiled library.
    # It was installed along with this .pm6 file, so it should be somewhere in
    # @*INC
    sub library {
        my $so = get-vars('')<SO>;
        my $libname = "libfoo$so";
        my $base = "lib/MyModule/$libname";
        for @*INC {
            if my @files = ($_.files($base) || $_.files("blib/$base")) {
                my $files = @files[0]<files>;
                my $tmp = $files{$base} || $files{"blib/$base"};

                # copy to a temp dir
                #
                # This is required because CompUnitRepo::Local::Installation stores the file
                # with a different filename (a number with no extension) that NativeCall doesn't
                # know how to load. We do this copy to fix the filename.
                $tmp.IO.copy($*SPEC.tmpdir ~ '/' ~ $lib);
                return $*SPEC.tmpdir ~ '/' ~ $lib;
            }
        }
        die "Unable to find library";
    }

    # we put 'is native(&library)' because it will call the function and resolve the
    # library at compile time, while we need it to happen at runtime (because
    # this library is installed *after* being compiled).
    sub foo() is native(&library) { * };

Functions
---------

### sub get-vars

```
sub get-vars(
    Str $destfolder
) returns Hash
```

Returns configuration variables. Effectively just a wrapper around $*VM.config, as the VM config variables are different for each backend VM.

### sub process-makefile

```
sub process-makefile(
    Str $folder, 
    %vars
) returns Mu
```

Takes '$folder/Makefile.in' and writes out '$folder/Makefile'. %vars should be the result of get-vars above.

### sub make

```
sub make(
    Str $folder, 
    Str $destfolder
) returns Mu
```

Calls get-vars and process-makefile for you to generate '$folder/Makefile', then runs your system's 'make' to build it.
