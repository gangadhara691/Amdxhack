> Bimanual teleop/runbook for dual SO101 leader/follower (left+right) with stable USB names.

## Identify stable device paths

Use by-id symlinks so `/dev/ttyACM*` renumbering doesn’t break you:

```bash
ls -l /dev/serial/by-id
```

Example mapping (replace with your actual serials):
- Left leader:  `/dev/serial/by-id/usb-1a86_USB_Single_Serial_5A68010289-if00`
- Left follower: `/dev/serial/by-id/usb-1a86_USB_Single_Serial_5A68010543-if00`
- Right leader: `/dev/serial/by-id/usb-1a86_USB_Single_Serial_5AE6084818-if00`
- Right follower: `/dev/serial/by-id/usb-1a86_USB_Single_Serial_5AE6054127-if00`

Optional: create udev aliases once:
```bash
sudo tee /etc/udev/rules.d/99-lerobot-dual.rules >/dev/null <<'EOF'
SUBSYSTEM=="tty", ATTRS{serial}=="5A68010289", SYMLINK+="ttyLEFT_LEADER"
SUBSYSTEM=="tty", ATTRS{serial}=="5A68010543", SYMLINK+="ttyLEFT_FOLLOWER"
SUBSYSTEM=="tty", ATTRS{serial}=="5AE6084818", SYMLINK+="ttyRIGHT_LEADER"
SUBSYSTEM=="tty", ATTRS{serial}=="5AE6054127", SYMLINK+="ttyRIGHT_FOLLOWER"
EOF
sudo udevadm control --reload-rules
sudo udevadm trigger
```
Then use `/dev/ttyLEFT_LEADER`, etc., instead of ACM numbers.

## Calibration files

Leader cal dir: `~/.cache/huggingface/lerobot/calibration/teleoperators/so101_leader/`

Follower cal dir: `~/.cache/huggingface/lerobot/calibration/robots/so101_follower/`

Ensure you have distinct IDs/files per device, e.g.:
- `left_leader.json`
- `right_leader.json`
- `left_follower.json`
- `right_follower.json`

If you only have `my_awesome_*` files, either recalibrate with the desired `--id` or copy/rename the correct file to the ID you will use.

## Calibrate (if needed)

Left leader (example):
```bash
lerobot-calibrate \
  --teleop.type=so101_leader \
  --teleop.port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5A68010289-if00 \
  --teleop.id=left_leader \
  --teleop.calibration_dir=~/.cache/huggingface/lerobot/calibration/teleoperators/so101_leader
```

Right leader:
```bash
lerobot-calibrate \
  --teleop.type=so101_leader \
  --teleop.port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5AE6084818-if00 \
  --teleop.id=right_leader \
  --teleop.calibration_dir=~/.cache/huggingface/lerobot/calibration/teleoperators/so101_leader
```

Left follower:
```bash
lerobot-calibrate \
  --robot.type=so101_follower \
  --robot.port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5A68010543-if00 \
  --robot.id=left_follower \
  --robot.calibration_dir=~/.cache/huggingface/lerobot/calibration/robots/so101_follower
```

Right follower:
```bash
lerobot-calibrate \
  --robot.type=so101_follower \
  --robot.port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5AE6054127-if00 \
  --robot.id=right_follower \
  --robot.calibration_dir=~/.cache/huggingface/lerobot/calibration/robots/so101_follower
```

## Bimanual teleop (both arms in one process)

If you have the bi-arm support in this branch, use `bi_so101_follower` + `bi_so101_leader` with both ports:

```bash
lerobot-teleoperate \
  --robot.type=bi_so101_follower \
  --robot.left_arm_port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5A68010543-if00 \
  --robot.right_arm_port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5AE6054127-if00 \
  --robot.id=dual_followers \
  --robot.calibration_dir=~/.cache/huggingface/lerobot/calibration/robots/so101_follower \
  --teleop.type=bi_so101_leader \
  --teleop.left_arm_port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5A68010289-if00 \
  --teleop.right_arm_port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5AE6084818-if00 \
  --teleop.id=dual_leaders \
  --teleop.calibration_dir=~/.cache/huggingface/lerobot/calibration/teleoperators/so101_leader
```

This uses:
- `left/right_*` IDs in their respective cal dirs.
- Stable by-id paths so ACM renumbering doesn’t matter.

If your calibration files use different IDs, adjust `--robot.id` / `--teleop.id` to match those filenames.

