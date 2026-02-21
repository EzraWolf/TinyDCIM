
WIP.

Toolchain Stack
- Code editor (VSCode)
- IIC-OSIC
    - XSchem
    - KLayout
    - IHP SG13G2 Open PDK


Setup (TENTATIVE)
1. Install Docker
2. Install UV for Python
2. Create toolchain root `$HOME/Toolchain/`
3. Install [IIC-OSIC-Tools](https://github.com/iic-jku/iic-osic-tools?tab=readme-ov-file) to toolchain root as `IIC-OSIC` - OpenPDK
   - good video to watch https://www.youtube.com/watch?v=l8kZUocmY5k
4. Install [OSS-CAD-Tools](https://github.com/YosysHQ/oss-cad-suite-build) to toolchain root as `OSS-CAD`
5. Setup paths (?) like `PDK`, `PDK_ROOT`, etc.
6. Run `./start_x.sh` "s" if already running
   - In the newly launched X terminal, run `iic-pdk` to see which PDKs you have installed
   - Run `sak-pdk` with either `ihp-sg13g2` (IHP), `sky130A` (Skywater) `gf180mcuD` (Global Foundaries) to automatically set the proper export paths
6. Run programs by typing their name. E.g., `klayout` and it will already have the proper PDK installed
   - good KLayout tutorial using IIC-OSIC tools https://www.youtube.com/watch?v=40qe4hXG6Kk&list=PLtzXUac_Lg4jstfqzxlK9bqxuxauoRiMO&index=4
