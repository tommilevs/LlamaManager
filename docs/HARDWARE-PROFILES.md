# Hardware profiles

Built-in immutable strategies:

- `auto`
- `vulkan`
- `cpu`
- `cuda`

The typo alias `vulcan` is accepted and normalized to `vulkan`.

External YAML profiles shipped in v0.1:

- `rx5500xt`
- `rtx4090`
- `rtx5060`
- `tesla-v100-32gb`

Profiles are matched conservatively by vendor/name substring. Multiple GPUs are retained in the inventory; the detector does not stop after the first CUDA or Vulkan device.

The host CPU is represented as a first-class accelerator with backend `cpu`, which allows CPU-only nodes to participate in placement decisions.
