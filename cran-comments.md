## Test environments

* local build Windows 11 R 4.5.2
* local build ubuntu 24.04.3 LTS, R 4.5.2
* GitHub Actions builds:
  * ubuntu-latest: 24.04.3 LTS, R-release (4.5.2)
  * ubuntu-latest: 24.04.3 LTS, R-oldrel (4.4.3)
  * ubuntu-latest: 24.04.3 LTS, R-devel (unstable 2025-11-30 r89082)
  * ubuntu-latest: 24.04.3 LTS, R-next (4.5.2 patched)
  * macOS-latest: Sequoia 15.7.1, R-release (4.5.2)
  * windows-latest: Server 2022 x64 Build 26100, R-release (4.5.2)
  * windows-oldrel: Server 2022 x64 Build 26100, R-release (4.4.3)
* win-builder builds:
  * release: windows server 2022 x64 Build 20348, R-release (4.5.2)
  * devel: windows server 2022 x64 Build 20358, R-devel (2025-11-30 r89082 ucrt)
  * oldrel: windows server 2022 x64 Build 20348, R-oldrel ()

## R CMD check results

* All builds:

  0 errors | 0 warnings | 0 notes

## Downstream dependencies

None.

## Additional comments

* This submission updates the package to version 0.2.5

  This was a very minor revision that updated the CITATION information
  to use the current format and updated R code to fix a new NOTE that 
  started appearing in R CMD check, about code like 
  `if (class(x) != "logical")`, which has potential to create problems
  because `class(x)` can return a vector. Replaced this with
  `if (! methods::is(x, "logical"))`, which will work even if `x` 
  belongs to  multiple classes.
