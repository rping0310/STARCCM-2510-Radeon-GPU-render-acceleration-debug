# STARCCM-2510-Radeon-GPU-render-acceleration-debug
Fix STAR-CCM+'s software-rendering fallback on AMD Radeon PRO W7900 (RDNA3) under Ubuntu 22.04. STAR-CCM+ ships an old bundled Mesa stack that doesn't recognize gfx1100 and an old libstdc++ that blocks the system's radeonsi DRI driver from loading.
