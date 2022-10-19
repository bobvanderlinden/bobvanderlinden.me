+++
title = "Customizing packages in Nix"
date = 2022-10-16

[taxonomies]
tags = ["nix", "nixos", "nixpkgs"]
+++

Nixpkgs provides a large number of packages that can be used in various ways.
In some cases you want to make modifications to a package.

In this post we'll go through various ways [how modified packages can be used in the Nix ecosystem](#using-modified-packages), [how packages can be modified](#modification-methods) and finally show some common use-cases:

* [Use a different version of a package](#use-different-version-of-a-package-in-nixos)
* [Use local source code for package](#use-local-source-code-for-a-package)
* [Use a different dependency for a single package](#use-a-different-dependency-for-a-single-package)
* [Apply a security patch system-wide](#apply-a-security-patch-system-wide)

<!-- more -->

## Introduction

In Nix packages are usually referred to as `pkgs.git`. This expression can be extended to allow modification of the package. For example, instead of `pkgs.git` you can also use `(pkgs.git.override { guiSupport = true; })`. This results in a new package where support for Git GUI is enabled. When using this expression, a package is built that includes the `git-gui` binary.

There are a couple of ways packages can be altered and there are multiple ways those altered packages can be used.

We'll first go through how packages can be used to get a sense where this fits in the Nix ecosystem.

## Using modified packages

Packages in Nix are referred from various places. Creating a modified package doesn't do much if nothing of the system is referring to the modified package. There are a number of places in practice where these packages can be placed in order to make them usable.

* [NixOS systemPackages](#nixos-systempackages) for the shell environment
* [NixOS package option](#nixos-package-option) for background services
* [Overlay](#overlay) for in-place package replacements and additions

### NixOS systemPackages

Using a custom package in `environment.systemPackages` will only change the system environment. Basically, it'll allow you to change the packages that you'd use in your terminal, but not anything else.

Changing:

```nix
{
  environment.systemPackages = [
    pkgs.git
  ];
}
```

To:

```nix
{
  environment.systemPackages = [
    (pkgs.git.override { guiSupport = true; })
  ];
}
```

will allow you to use the Git GUI tools (`git-gui`) in your terminal. It _doesn't_ alter packages or NixOS modules that _depend_ on Git. This is an important distinction, as it only affects the environment (applications available in `PATH`) and doesn't rebuild any other packages.

### NixOS package option

Some packages are only used in background services, so you won't interact with them directly. In NixOS these are often configured with `services.*.enable` options.

These options usually also have an option to change the package for the service, named `services.*.package`.

For example, pipewire is a service to handle audio and video streams on a system. It is enabled on NixOS using:

```nix
{
  services.pipewire.enable = true;
}
```

To override the pipewire package being used, the following option can be added:

```nix
{
  services.pipewire.package = pkgs.pipewire.override { x11Support = false; };
}
```

In this case `pipewire` is compiled without x11 support. The resulting package is used as the pipewire background service, which will be configured by NixOS in systemd.

Just like with `systemPackages`, the altered package is only used in the NixOS system and packages that depend on pipewire will still refer to the original pipewire package.

### Overlay

Overlays in Nix allow extending Nixpkgs. Like the name suggests, it overlays your package definitions on top of Nixpkgs. Usually this means that you can alter what packages are available in the `pkgs` variable, allowing you to add or overwrite packages.

With overlays you can replace an existing package system-wide. That means overwriting an existing package with your own will not only change your system, but can also change all packages that depend on the original package.

This can be useful when you want to for instance patch a library that is used by other software. It does however require all depending packages to be rebuild. Nix does these rebuilds seamlessly and automatically, but it does cost time.

Also a word of warning: overlays can become confusing. You generally want to be aware of overwritten packages in overlays. It is not obvious when referring to a package `pkgs.git` that it is different from what is in Nixpkgs. Especially when overwriting packages like `openssl`, which `git` and many other packages depends on. With great power comes great responsibility!

On NixOS you can use [the `nixpkgs.overlays` option](https://search.nixos.org/options?channel=22.05&show=nixpkgs.overlays&query=overlays):

```nix
{
  nixpkgs.overlays = [
    (final: prev: {
      pipewire = prev.pipewire.override { x11Support = false; };
    })
  ];
}
```

When creating such an overlay, it is not needed anymore to specify the override in `systemPackages` nor `services.*.package`, as `pkgs.pipewire` will only refer to this overridden package.

Note that the overlay is defined as a function with the arguments `final` and `prev`.

`prev` refers to the previous layer, the underlying one. In this case that is `nixpkgs`. We take the original package from `nixpkgs` using `prev.pipewire` and alter that package. With `pipewire = ...;` we overwrite the orignal pipewire package in succeeding layers, which eventually results in the change in `pkgs`.

`final` refers to the top-level overlay, which includes our overwritten packages. If we want to refer to a another package that we have overridden, then we can refer to `final`.

More information about overlays can be found in [the Nixpkgs manual](https://nixos.org/manual/nixpkgs/stable/#chap-overlays).

It is also possible to introduce a new attribute for custom packages, so that you can refer to this attribute in other parts of your configuration. For instance:

```nix
{
  nixpkgs.overlays = [
    (final: prev: {
      git-with-gui = prev.git.override { guiSupport = true; };
    })
  ];
}
```

will allow referring to `pkgs.git-with-gui` where-ever you have access to pkgs. For instance in NixOS:

```nix
{
  environment.systemPackages = [ pkgs.git-with-gui ];
}
```

Or even in Nix CLI:

```sh
nix profile install git-with-gui
```

Note that even though we have a new attribute name in the above example, the _name of the package_ didn't change. The name is part of the derivation. The package is called `git`, so the Nix store path is still `/nix/store/*-git`. When using `.override` this part of the derivation would not change. The name can only be changed using `.overrideAttrs`.

## Modification methods

Now that we know where we can use modified packages, we will dive into two ways to modify a package: `override` and `overrideAttrs`.

### `<pkg>.override`

Each package in nixpkgs provide a number of options that the package depends on.
These options are usually other packages that the package depends on, but can also be pre-defined configuration options.

Documentation and discoverability of these is often quite limited. That's why it is usually needed to look into the source of the package in the nixpkgs repository.

Using http://search.nixos.org/packages it is possible to find a package where you can click through to the source code.

At the top of package files, there are the package options. For instance, for git, we can find them [here](https://github.com/NixOS/nixpkgs/blob/78a37aa630faa41944060a966607d4f1128ea94b/pkgs/applications/version-management/git-and-tools/git/default.nix#L1-L22). This includes the option `guiSupport` that we've seen earlier.

Overriding these options usually alters the configuration flags (like `guiSupport`) or you can change one of the package dependencies. For instance, overriding both `guiSupport` and the version of `curl` to use:

```nix
(git.override { guiSupport = true; curl = curlFull; })
```

### `<pkg>.overrideAttrs`

In addition to the pre-defined options that a package provides, it is possible to change attributes of the derivation itself. For instance, the source of the package, the patches it should use, the build steps it should execute. All of these aren't usually provided by the package as options, but they can be altered or extended using `overrideAttrs`.

For instance, adding a patch to `curl` can be done like:

```nix
(curl.overrideAttrs (previousAttrs: {
  patches = previousAttrs.patches ++ [ ./my-patch.patch ];
}))
```

Here `previousAttrs` refers to the attributes that were used by the original curl package. We can use these attributes to not just overwrite, but also extend existing attributes. In this case, add a patch the list of patches that `curl` is already using. The patch in this case is a local file called `my-patch.patch`, which should reside right next to the `.nix` file where code is used.

## Common use-cases

With the above knowledge we can tackle a number of common use-cases.

### Use different version of a package in NixOS

```nix
{
  services.xserver.windowManager.i3.package = pkgs.i3.overrideAttrs (previousAttrs: {
    name = "i3-next";
    src = pkgs.fetchFromGitHub {
      owner = "i3";
      repo = "i3";
      rev = "81287743869a5bdec4ffc0c1e6d1f8fd33920bcb";
      hash = pkgs.lib.fakeHash;
    };
  });
}
```

This overrides the `i3` package with a version based on https://github.com/i3/i3/commit/81287743869a5bdec4ffc0c1e6d1f8fd33920bcb. `pkgs.lib.fakeHash` is a placeholder for the actual hash of the retrieved directory. Upon building Nix will ask to replace it with the actual hash that Nix calculated.

The `name = "i3-next"` is also set. This is to make sure the new package name doesn't equal the original package name (`i3-X.X.X`).

### Use local source code for a package

```nix
{
  environment.systemPackages = [ pkgs.myfortune ];

  nixpkgs.overlays = [
    (final: prev: {
      myfortune = prev.fortune.overrideAttrs (previousAttrs: {
        src = ./fortune-src;
      });
    })
  ];
}
```

This creates a new attribute `myfortune` that uses the build steps from `fortune` to build source code from a local directory `./fortune-src` into a package. It makes the package available through `environment.systemPackages`, so that it is available in the terminal.

### Use a different dependency for a single package

```nix
{
  nixpkgs.overlays = [
    (final: prev: {
      maven-jdk8 = prev.maven.override {
        jdk = final.jdk8;
      };
    })
  ];
}
```

This creates a new attribute `maven-jdk8` that builds and runs maven explicitly under `jdk8`. Notice that `final.jdk8` is used, so that other overlays may potentially overwrite `jdk8`. Also note that `final.maven` is _not_ used, because that would refer to this package, causing an infinite loop during Nix evaluation.

### Apply a security patch system-wide

```nix
{
  nixpkgs.overlays = [
    (final: prev: {
      openssl = prev.openssl.overrideAttrs (previousAttrs: {
        patches = previousAttrs ++ [
          (fetchpatch {
            name = "CVE-2021-4044.patch";
            url = "https://git.openssl.org/gitweb/?p=openssl.git;a=patch;h=758754966791c537ea95241438454aa86f91f256";
            hash = pkgs.lib.fakeHash;
          })
        ];
      });
    })
  ];
}
```

This overwrites `openssl` with a patched version. The patch itself is fetched from OpenSSL's own git repository. Overwriting `openssl` in an overlay will rebuild and test all packages that depend on OpenSSL as well, so in this case can lead to high build times.

## Conclusion

Hopefully this gave an overview of the ways to customize packages in NixOS and how to use this in practice.

Let me know if there were any use-cases that your were missing. I'll update the article
