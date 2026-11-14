# Third-Party Notices

This source tree retains the original copyright and license notices of its dependencies.

| Component | Location | License notice |
| --- | --- | --- |
| HTSlib 1.20, including bundled htscodecs | `third_party/htslib-1.20/` | `LICENSE` and `htscodecs/LICENSE.md` |
| Host libdeflate 1.20 | `third_party/libdeflate-1.20/` | `COPYING` |
| Adapted CPE libdeflate | `swbam-sdk/slave/slave_zlib/` | `COPYING` and source-file notices |
| CLI11 2.5.0 | `swbam-app/third_party/CLI11.hpp` | BSD-3-Clause notice embedded in the header |

The HTSlib and CPE codec adaptations are carried over from the source project; they must not be replaced silently by unmodified upstream builds.
The license for the SWBAM-authored project as a whole has not been selected in this packaging step.
