
<!--
To preview this README.md file locally:
gh-markdown-preview -p 8000 --markdown-mode README.md
-->

## Access to the Olinpe data

The data for the years 2022-2024 where sent to us by email direclty from
Jeremy Blatrix (Atelier de la Donnée).
The data used for the analysis has been downloaded from an email sent by Mikel
Redin Hurtado on 01/21/26 and stored in `data/archive/DataRecoded.xlsx`.

After a quick exploration with `visidata`, I realised that the Excel files sent
were not anonymised... So I  will remove the columns "NOM", "PRENOM", "JNAIS"
before importing the data in R.

Conversion using `in2csv` did not work properly (same with visidata, there
seems to be an encoding problem), but I could convert the data to CSV from
Libreoffice.

I used `csvcut` to quickly remove columns that allow the identification of an
individual:

```bash
# %% Remove columns 'NOM', 'PRENOM', 'JNAIS' for each year

remove_cols() {
    csvcut -x -c "1-4,7-9,11-110" "data/archive/$1" | \
        tee "data/raw/$1"
}

remove_cols "Enquete_olinpe_2022.csv"
remove_cols "Enquete_olinpe_2023.csv"
remove_cols "Enquete_olinpe_2024.csv"

```



## Reproducible setup

### Project structure

```bash
# %% File structure

# data/ directory

mkdir -p data/{archive,raw,processed}

# .config/ directory

mkdir -p .config/rix

# R/ directory
mkdir R
```

### Computing environment

* Open an R shell with the `rix` package using `nix-shell` in the `.config/rix`
subdirectory:

```bash
nix-shell -p R rPackages.rix
```

* Get the latest available date for R and Biodonductor releases with
`rix::available_dates()`.

→ `"2026-03-09`


* Execute the cell below to create the `gen-env.R` that will be used to generate
a `default.nix` file for our project using the `rix` package.

```bash
# %% Create gen-env.R

cat << EOF > .config/rix/gen-env.R
library(rix)

rix(
  date = "2026-03-09",
  r_pkgs = c("here", "tidyverse", "bayesplot", "brms", "posterior"),
  py_conf = list(
      py_version = "3.13"
  ),
  git_pkgs = list(
    list(
      package_name = "cmdstanr",
      repo_url = "https://github.com/stan-dev/cmdstanr",
      commit = "da99e2ba954658bdad63bffb738c4444c33a4e0e"
    ),
    list(
      package_name = "httpgd",
      repo_url = "https://github.com/nx10/httpgd",
      commit = "dd6ed3a687a2d7327bb28ca46725a0a203eb2a19"
    ),
    list(
      package_name = "hrbrthemes",
      repo_url = "https://github.com/hrbrmstr/hrbrthemes",
      commit = "d3fd02949fc201c6db616ccaffbb9858aec6fd2b"
    )
  ),
  system_pkgs = c("csvkit", "git"),
  ide = "radian",
  project_path = ".",
  shell_hook = "
      alias vm='export NVIM_APPNAME='\''nvim-minimal'\''; nvim'
  ",
  overwrite = TRUE
)
EOF
```

* Run the script to get the `default.nix` file:

```bash
# %% Create `.config/rix/default.nix`

Rscript gen-env.R
```

* Use the content in `.config/rix/default.nix` to create the files `flake.nix` and
`shell.nix` at the root of the repository:

```bash
# %% Create Nix flake files

cat << EOF > flake.nix
{
  description = "Reproducible data analysis shell";

  inputs = {
    nixpkgs.url = "https://github.com/rstats-on-nix/nixpkgs/archive/2026-03-09.tar.gz";
  };

  outputs = { self, nixpkgs }:
  let
    pkgs = nixpkgs.legacyPackages."x86_64-linux";
  in
  {

    devShells."x86_64-linux".default =
      (import ./shell.nix { inherit pkgs; }).shell;
  };
}
EOF

cat << 'EOF' > shell.nix
{ pkgs ? import <nixpkgs> {} }:

let
  rpkgs = builtins.attrValues {
    inherit (pkgs.rPackages) 
      bayesplot
      brms
      here
      posterior
      tidyverse;
  };
 
    cmdstanr = (pkgs.rPackages.buildRPackage {
      name = "cmdstanr";
      src = pkgs.fetchgit {
        url = "https://github.com/stan-dev/cmdstanr";
        rev = "da99e2ba954658bdad63bffb738c4444c33a4e0e";
        sha256 = "sha256-wXfOxBexnuL83fvCM+6qv6d7UhTLGq+Xhvja3lRfQpI=";
      };
      propagatedBuildInputs = builtins.attrValues {
        inherit (pkgs.rPackages) 
          checkmate
          data_table
          jsonlite
          posterior
          processx
          R6
          withr
          rlang;
      };
    });

    hrbrthemes = (pkgs.rPackages.buildRPackage {
      name = "hrbrthemes";
      src = pkgs.fetchgit {
        url = "https://github.com/hrbrmstr/hrbrthemes";
        rev = "d3fd02949fc201c6db616ccaffbb9858aec6fd2b";
        sha256 = "sha256-BfclWuD7JsHrscAXO8FmS8239TTYywGcQBp0BnSggYs=";
      };
      propagatedBuildInputs = builtins.attrValues {
        inherit (pkgs.rPackages) 
          ggplot2
          scales
          extrafont
          magrittr
          gdtools;
      };
    });

    httpgd = (pkgs.rPackages.buildRPackage {
      name = "httpgd";
      src = pkgs.fetchgit {
        url = "https://github.com/nx10/httpgd";
        rev = "dd6ed3a687a2d7327bb28ca46725a0a203eb2a19";
        sha256 = "sha256-vs6MTdVJXhTdzPXKqQR+qu1KbhF+vfyzZXIrFsuKMtU=";
      };
      propagatedBuildInputs = builtins.attrValues {
        inherit (pkgs.rPackages) 
          unigd
          cpp11
          AsioHeaders;
      };
    });
   
 
  pyconf = builtins.attrValues {
    inherit (pkgs.python313Packages) 
      pip
      ipykernel;
  };
   
  system_packages = builtins.attrValues {
    inherit (pkgs) 
      csvkit
      git
      glibcLocales
      nix
      python313
      R;
  };
 
  wrapped_pkgs = pkgs.radianWrapper.override {
    packages = [ cmdstanr httpgd hrbrthemes rpkgs  ];
  };
 
  shell = pkgs.mkShell {
    LOCALE_ARCHIVE = if pkgs.stdenv.hostPlatform.system == "x86_64-linux" then "${pkgs.glibcLocales}/lib/locale/locale-archive" else "";
    LANG = "en_US.UTF-8";
    LC_ALL = "en_US.UTF-8";
    LC_TIME = "en_US.UTF-8";
    LC_MONETARY = "en_US.UTF-8";
    LC_PAPER = "en_US.UTF-8";
    LC_MEASUREMENT = "en_US.UTF-8";
    RETICULATE_PYTHON = "${pkgs.python313}/bin/python";

    buildInputs = [ cmdstanr httpgd hrbrthemes rpkgs pyconf system_packages wrapped_pkgs ];
    shellHook = ''
    
      alias vm='export NVIM_APPNAME='''nvim-minimal'''; nvim'
  
  '';
  }; 
in
  {
    inherit pkgs shell;
  }
EOF
```

<!--
NB: Executing the code cell using Vim slime inserts backslashes that break the
creation of the Nix shell. Instead, use CTRL-x CTRL-e to paste command in Vim
buffer directly from shell.
-->

* Run the Nix development shell with `nix develop` and launch `radian` to
  start an R session.


### `targets` for reproducible pipeline

```r
# %% Initialise targets pipeline

targets::use_targets()

```

```bash
# Make the `_targets.R` file writable and executable  
# (not sure why restriction to read-only)

chmod +wx _targets.R
```

Visualise pipeline with `targets::tar_visnetwork()`.
to fix browser opening issue:

```bash

echo 'options(browser = "/usr/local/bin/firefox")' > .Rprofile

```

### Quarto website

Start from website template:

```bash

quarto create project website .

# Title:
# Données Olinpe CD78

rm about.qmd

```

Edit `_quarto.yml`:

```yaml
project:
  type: website
  output-dir: docs

website:
  title: "Données Olinpe CD78"
  navbar:
    left:
      - href: index.qmd
        text: Home
      - href: analyses/01-data-explo.qmd
        text: Exploration des données

format:
  html:
    theme:
      - cosmo
      - brand
    css: styles.css
    toc: true

execute:
  freeze: auto
```

Commit changes before deploying website:

```bash
git add docs
git commit -m "Update site"
git push
```

Change repository settings to configure Github pages:
* Settings → Pages
* Source = Deploy from branch
* Branch = main
* Folder = /docs

On the home page of the
[Github repository](https://github.com/hugomell/paper-redin-hurtado-2026),
update the `About -> Website` settings to check "Use your GitHub Pages
website".



<!-- Graveyard

```bash
# %% Convert raw data from Excel spreadsheet to csv

convert_to_csv() {
    output_name="$(echo $1 | cut -f1 -d'.').csv"
    in2csv --blanks --sheet "$2" \
        data/archive/"$1" | \
        csvcut -x -c "$3" | \
        tee "data/raw/$output_name" | csvlook --blanks | less -S
}

excel_sheet="BASE_TOTALE"
cols_selection="1-4,7-9,11-110"

# Olinpe data 2022
excel_file="Enquete_olinpe_2023.xlsx"
convert_to_csv "$excel_file" "$excel_sheet" "$cols_selection"

```


-->
