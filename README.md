# treefmt-nix — Fast and convenient multi-file formatting with Nix

[treefmt](https://numtide.github.io/treefmt) combines file formatters for multiple programming languages so that you
can format all your project files with a single command. With `treefmt-nix` you can specify `treefmt` build options,
dependencies and config in one place, conveniently managed by [Nix](https://nixos.org/).

`treefmt-nix` automatically installs and configures the desired formatters as well as `treefmt` for you and integrates
nicely into your Nix development environments.
It comes with sane, pre-crafted [formatter-configs](https://github.com/numtide/treefmt-nix/tree/main/programs)
maintained by the community; each config corresponds to a section that you would normally add to the `treefmt`
config file `treefmt.toml`. 

Take a look at the already [supported formatters](#supported-programs) for Python, Rust, Go, Haskell and more.
 
## Integration into Nix

### Nix classic without flakes

To run `treefmt-nix` with nix-classic, import the repo using [`niv`](https://github.com/nmattia/niv):
```
$ niv add numtide/treefmt-nix
```
Alternatively, you can download the source and run `nix-build` in the project root directory:
```
$ nix-build
```

The command will return the helper functions which will be later used to produce a derivation from the specified `treefmt-nix` configuration.

After you installed treefmt-nix, specify the formatter configuration. For instance, this one is for formatting terraform files:

```nix
# myfile.nix
{ system ? builtins.currentSystem }:
let
  nixpkgsSrc = builtins.fetchTarball "https://github.com/NixOS/nixpkgs/archive/refs/heads/nixos-unstable.tar.gz";
  treefmt-nixSrc = builtins.fetchTarball "https://github.com/numtide/treefmt-nix/archive/refs/heads/master.tar.gz";
  nixpkgs = import nixpkgsSrc { inherit system; };
  treefmt-nix = import treefmt-nixSrc;
in
treefmt-nix.mkWrapper nixpkgs {
  # Used to find the project root
  projectRootFile = ".git/config";
  # Enable the terraform formatter
  programs.terraform.enable = true;
  # Override the default package
  programs.terraform.package = nixpkgs.terraform_1;
  # Override the default settings generated by the above option
  settings.formatter.terraform.excludes = [ "hello.tf" ];
}
```
It's a good practice to place the configuration file in the project root directory.

Next, execute this command:
```
$ nix-build myfile.nix
```

This command returns a derivation that contains a `treefmt` binary at `./result/bin/treefmt` in your current directory. The file is actually a symlink to the artifact in `/nix/store`. 

`treefmt.toml` in this case isn't generated: the binary is wrapped with the config.


### Flakes

Running treefmt-nix with flakes isn't hard. The library is exposed as the `lib` attribute:

```nix
# flake.nix
{
  inputs.treefmt-nix.url = "github:numtide/treefmt-nix";

  outputs = { self, nixpkgs, systems, treefmt-nix }:
    let
      # Small tool to iterate over each systems
      eachSystem = f: nixpkgs.lib.genAttrs (import systems) (system: f nixpkgs.legacyPackages.${system});

      # Eval the treefmt modules from ./treefmt.nix
      treefmtEval = eachSystem (pkgs: treefmt-nix.lib.evalModule pkgs ./treefmt.nix);
    in
    {
      # for `nix fmt`
      formatter = eachSystem (pkgs: treefmtEval.${pkgs.system}.config.build.wrapper);
      # for `nix flake check`
      checks = eachSystem (pkgs: {
        formatting = treefmtEval.${pkgs.system}.config.build.check self;
      });
    };
}
```

And also add the `treefmt.nix` file (or put the content inline if you prefer):
```nix
# treefmt.nix
{ pkgs, ... }:
{
  # Used to find the project root
  projectRootFile = "flake.nix";
  # Enable the terraform formatter
  programs.terraform.enable = true;
  # Override the default package
  programs.terraform.package = pkgs.terraform_1;
  # Override the default settings generated by the above option
  settings.formatter.terraform.excludes = [ "hello.tf" ];
}
```
This file is also the place to define all the treefmt parameters like includes, excludes and formatter options. 

After specifying the flake, run [`nix fmt`](https://nixos.org/manual/nix/stable/command-ref/new-cli/nix3-fmt.html):
```
$ nix fmt
```
Nix-fmt is a tool to format all nix files in the project, but with the specified flake, it starts treefmt-nix and formats your project. 

You can also run `nix flake check` (eg: in CI) to validate that the project's
code is properly formatted.

### Flake-parts

This flake exposes a [flake-parts](https://flake.parts/) module as well. To use it:

1. Add `inputs.treefmt-nix.flakeModule` to the `imports` list of your `flake-parts` call.

3. Add `treefmt = { .. }` (containing the configuration above) to your `perSystem`.

5. Add `config.treefmt.build.wrapper` to the `nativeBuildInputs` of your `devShell`. This will make the `treefmt` command available in the shell using the specified configuration.

    You can also use `config.treefmt.build.programs` to get access to the individual programs, which could be useful to provide them to your IDE or editor.

    For an example, see [haskell-template](https://github.com/srid/haskell-template)'s `flake.nix`.
    
See [this page](https://haskell.flake.page/treefmt) for details.

## Configuration
While dealing with `treefmt` outside of `nix`, the formatter configuration is specified in a `toml` format. On the contrary, with `nix`, you write in with a nix syntax like this:
```nix
  # Used to find the project root
  projectRootFile = ".git/config";
  # Enable the terraform formatter
  programs.terraform.enable = true;
  # Override the default package
  programs.terraform.package = nixpkgs.terraform_1;
  # Override the default settings generated by the above option
  settings.formatter.terraform.excludes = [ "hello.tf" ];
```

**Options:** 
* `Project root file` is the git file of the project which you plan to format. 
* The option `programs.terraform.enable` enables the needed formatter. You can specify as many formatter as you want. For instance:
```
programs.terraform.enable = true;
programs.gofmt.enable = true;
```
* The option `programs.terraform.package` allows you to use a particular build/version of the specified formatter.
* By setting`settings.formatter.terraform.excludes` you can mark the files which should be excluded from formatting. You can also specify other formatter options or includes this way. 

For detailed description of the options, refer to the `treefmt` [documentation](https://numtide.github.io/treefmt/treefmt-configuration.html). 

## Project structure

This repo contains a top-level `default.nix` that returns the library helper functions. 

* `mkWrapper` is the main function which wraps treefmt with the needed configuration.
* `mkConfigFile`
* `evalModule`
* `all-modules`

## Supported programs
`treefmt-nix` currently supports the following formatters:

<!-- `> ls programs/*.nix | grep -v default.nix | cut -d '.' -f 1 | cut -d / -f 2 | LC_ALL=C sort | sed -e 's/^/* /'` -->
<!-- BEGIN mdsh -->
* alejandra
* beautysh
* black
* buildifier
* cabal-fmt
* clang-format
* cue
* deadnix
* deno
* dhall
* dprint
* elm-format
* erlfmt
* fnlfmt
* gofmt
* gofumpt
* google-java-format
* hclfmt
* hlint
* isort
* ktfmt
* ktlint
* leptosfmt
* mdformat
* mdsh
* mix-format
* muon
* mypy
* nixfmt
* nixpkgs-fmt
* ocamlformat
* ormolu
* php-cs-fixer
* prettier
* protolint
* purs-tidy
* ruff
* rufo
* rustfmt
* scalafmt
* shellcheck
* shfmt
* statix
* stylish-haskell
* stylua
* taplo
* terraform
* yamlfmt
* zprint
<!-- END mdsh -->

For non-Nix users, you can also find the generated examples in the
[./examples](./examples)
folder.

### Adding new formatters

PRs to add new formatters are welcome!

* The formatter should conform to the [formatter specifications](https://numtide.github.io/treefmt/formatters-spec.html).
* This is not the place to debate formatting preferences. Please pick defaults that are standard in your community -- for instance, python is usually indented with 4 spaces, so don't add a python formatter with 2 spaces as the default.

In order to add a new formatter do the following things:

1. Create a new entry in the ./programs/ folder.
2. Run ./bors.toml.sh to update the bors.toml file.
3. Run ./examples.sh to update the ./examples folder.
4. To test the program:
    1. Extend the project's `./treefmt.nix` file (temporarilt)
    2. Add a bunch of sources in this repo
    3. Run `nix develop -c treefmt`
    4. Once this is good, revert those changes.
5. Submit the PR!

## Commercial support

Looking for help or customization?

Get in touch with Numtide to get a quote. We make it easy for companies to
work with Open Source projects: <https://numtide.com/contact>

## License

All the code and documentation is licensed with the MIT license. 
