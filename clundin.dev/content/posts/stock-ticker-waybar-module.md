+++
date = '2025-06-28T12:14:55-07:00'
draft = false
title = 'Stock Ticker Waybar Module for NixOS'
+++

# Introduction

This post outlines a simple approach for adding a stock ticker to Waybar, specifically for a NixOS setup.

Once completed, your final result will look like the following:

![Picture](/stock-ticker-waybar/example.png)

# Pre-requisites

Before you start, you will need the following:

- An [FMP API Key](https://site.financialmodelingprep.com/developer/docs)
- A CLI that can fetch a ticker price.
  - In this post we will use [stock-ticker](https://github.com/clundin55/stock-ticker), a simple Rust CLI.

# Add stock ticker CLI

The waybar module will depend on the stock-ticker CLI.

## Add CLI to flake.nix

Add the CLI as a flake input to your system.

```nix
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";
    stock-ticker.url = "github:clundin55/stock-ticker";
  };
  ...
}
```

## Pass stock-ticker to configuration.nix

Add `stock-ticker` as a `specialArgs` so it can be passed to `configuration.nix` as a parameter.

```nix
{
  outputs =
    inputs@{ nixpkgs, home-manager, stock-ticker, ... }:
    my-system = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      specialArgs = {
        stock-ticker = stock-ticker.packages."x86_64-linux".default;
      };
      modules = [
        ./configuration.nix
        ...
      ];
    };
}
```

Finally, run `$ nix flake update stock-ticker` to update the system flake.lock.

## Add stock-ticker as a system package

Now we will modify **configuration.nix** to install stock-ticker as a system-wide package.

```nix
{ config, pkgs, stock-ticker, ... }:
{
  environment.systemPackages = with pkgs; [
    stock-ticker
  ];
}
```

Rebuild the system, and then check that the stock-ticker CLI is installed.

```bash
$ nixos-rebuild --switch
$ stock-ticker --h
Usage: stock-ticker --tickers <TICKERS>

Options:
  -t, --tickers <TICKERS>  A comma-separated list of stock ticker symbols
  -h, --help               Print help
  -V, --version            Print version
```

# Creating a wrapper script for Waybar

Now that the stock-ticker CLI is set up, we want to create a wrapper script that can be used by Waybar. 
This simplifies the Waybar configuration and lets us easily customize the Waybar module.

Next, introduce a `stock-price.sh` bash script by adding it to `environment.systemPackages`. This script will provide the FMP API key and specify which tickers to monitor.

```nix

{ config, pkgs, stock-ticker, ... }:
{
  environment.systemPackages = with pkgs; [
    stock-ticker
    ((pkgs.writeScriptBin "stock-price.sh" ''
    #!${pkgs.bash}/bin/bash

    set -eu
    export PMP_KEY=$(pass show Programming/pmp-key)
    stock-ticker --tickers GOOG
    ''))
  ];
}
```

This example script uses GNU pass to safely store the FMP API key. There are many ways to approach this but be sure to avoid committing the plain text API key to your NixOS configuration!

Now that the script is added, rebuild your system and check that it works.

```bash
$ nixos-rebuild --switch
$ stock-price.sh
GOOG $178.27 (3.84)
```

# Stock Ticker Waybar Module

Finally, we can now tie these pieces together and create our Waybar module.

Add the following JSON to your existing Waybar configuration and then restart Waybar.
```json
{
    "modules-right": ["custom/stock"],
    "custom/stock": {
        "exec": "stock-price.sh",
        "format": "{}",
        "interval": 86400
    },
}
```

# Summary

You can find my complete NixOS configuration on [GitHub](https://github.com/clundin55/nixos). All the changes described in this post are contained in this [commit](https://github.com/clundin55/NixOS/commit/5dbc560258f4dabba5d4c24ea15077073730e7c5).

