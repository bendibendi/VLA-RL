# 准备工作
```bash
# 查看串口
ls /dev/ttyACM*
# 给串口权限
sudo chmod 666 /dev/ttyACM*
# 查看摄像头
lerobot-find-cameras opencv
```

# 机械臂标定
```bash
# 左从
lerobot-calibrate \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM0 \
  --robot.id=bi_so101_follower_left

# 左主
lerobot-calibrate \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM2 \
  --teleop.id=bi_so101_leader_left

# 右从
lerobot-calibrate \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM1 \
  --robot.id=bi_so101_follower_right

# 右主
lerobot-calibrate \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM3 \
  --teleop.id=bi_so101_leader_right

```


# 遥操作：
```bash
# 左臂
lerobot-teleoperate \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM0\
  --robot.id=bi_so101_follower_left \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM2 \
  --teleop.id=bi_so101_leader_left

# 右臂
lerobot-teleoperate \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM1 \
  --robot.id=bi_so101_follower_right \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM3 \
  --teleop.id=bi_so101_leader_right

# 双臂
lerobot-teleoperate \
  --robot.type=bi_so_follower \
  --robot.id=bi_so101_follower \
  --robot.left_arm_config.port=/dev/ttyACM0 \
  --robot.right_arm_config.port=/dev/ttyACM1 \
  --teleop.type=bi_so_leader \
  --teleop.id=bi_so101_leader \
  --teleop.left_arm_config.port=/dev/ttyACM2 \
  --teleop.right_arm_config.port=/dev/ttyACM3

# 双臂+摄像头 注意摄像头端口号
lerobot-teleoperate \
  --robot.type=bi_so_follower \
  --robot.id=bi_so101_follower \
  --robot.left_arm_config.port=/dev/ttyACM0 \
  --robot.right_arm_config.port=/dev/ttyACM1 \
  --robot.left_arm_config.cameras='{wrist: {type: opencv, index_or_path: 2, width: 640, height: 480, fps: 30, fourcc: "MJPG", backend: 200}}' \
  --robot.right_arm_config.cameras='{wrist: {type: opencv, index_or_path: 4, width: 640, height: 480, fps: 30, fourcc: "MJPG", backend: 200}, front: {type: opencv, index_or_path: 6, width: 640, height: 480, fps: 30, fourcc: "MJPG", backend: 200}}' \
  --teleop.type=bi_so_leader \
  --teleop.id=bi_so101_leader \
  --teleop.left_arm_config.port=/dev/ttyACM2 \
  --teleop.right_arm_config.port=/dev/ttyACM3 \
  --display_data=true
```

# 录制数据 注意端口号
```bash
# 录制
lerobot-human-inloop-record \
  --robot.type=bi_so_follower \
  --robot.id=bi_so101_follower \
  --robot.left_arm_config.port=/dev/ttyACM0 \
  --robot.right_arm_config.port=/dev/ttyACM1 \
  --robot.left_arm_config.cameras='{wrist: {type: opencv, index_or_path: 2, width: 640, height: 480, fps: 30, fourcc: "MJPG", backend: 200}}' \
  --robot.right_arm_config.cameras='{wrist: {type: opencv, index_or_path: 4, width: 640, height: 480, fps: 30, fourcc: "MJPG", backend: 200}, front: {type: opencv, index_or_path: 6, width: 640, height: 480, fps: 30, fourcc: "MJPG", backend: 200}}' \
  --teleop.type=bi_so_leader \
  --teleop.id=bi_so101_leader \
  --teleop.left_arm_config.port=/dev/ttyACM2 \
  --teleop.right_arm_config.port=/dev/ttyACM3 \
  --dataset.repo_id=aiden/blocks_to_box_v2 \
  --dataset.single_task="Pick up the objects and put them into the box." \
  --dataset.num_episodes=40 \
  --dataset.episode_time_s=200 \
  --dataset.reset_time_s=10 \
  --dataset.push_to_hub=false \
  --dataset.vcodec=h264 \
  --play_sounds=false \
  --display_data=true

# 查看
lerobot-dataset-report --dataset aiden/blocks_to_box_test
lerobot-dataset-viz \
  --repo-id aiden/blocks_to_box_test \
  --episode-index 0 \
  --display-compressed-images=false
```




# 训练
```bash
# 双卡
HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1 CUDA_VISIBLE_DEVICES=0,1 accelerate launch \
  --multi_gpu --num_processes=2 --num_machines=1 --main_process_port=0 \
  --mixed_precision=bf16 --dynamo_backend=no \
  $(which lerobot-train) \
  --dataset.repo_id=aiden/blocks_to_box_v2 \
  --policy.type=pi05 \
  --policy.pretrained_path=/root/workspace/VLA-RL/checkpoints/lerobot/pi05_base \
  --policy.device=cuda \
  --policy.dtype=bfloat16 \
  --policy.gradient_checkpointing=true \
  --batch_size=4 \
  --steps=30000 \
  --log_freq=10 \
  --save_freq=10000 \
  --eval_freq=0 \
  --output_dir=/root/workspace/VLA-RL/outputs/train/blocks_to_box_v2_pi05_full_bs2effective_30k \
  --job_name=blocks_to_box_v2_pi05_full_bs2effective_30k \
  --wandb.enable=false \
  --policy.push_to_hub=false

# 单卡
HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1 CUDA_VISIBLE_DEVICES=0 \
accelerate launch \
  --num_processes=1 --num_machines=1 --main_process_port=0 \
  --mixed_precision=bf16 --dynamo_backend=no \
  $(which lerobot-train) \
  --dataset.repo_id=aiden/blocks_to_box_v2 \
  --policy.type=pi05 \
  --policy.pretrained_path=/root/workspace/VLA-RL/checkpoints/lerobot/pi05_base \
  --policy.device=cuda \
  --policy.dtype=bfloat16 \
  --policy.gradient_checkpointing=true \
  --batch_size=4 \
  --steps=30000 \
  --log_freq=10 \
  --save_freq=10000 \
  --eval_freq=0 \
  --output_dir=/root/workspace/VLA-RL/outputs/train/blocks_to_box_v2_pi05_full_bs2effective_30k_1gpu_save10k \
  --job_name=blocks_to_box_v2_pi05_full_bs2effective_30k_1gpu_save10k \
  --wandb.enable=false \
  --policy.push_to_hub=false


# 单卡继续训练
HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1 CUDA_VISIBLE_DEVICES=0 \
accelerate launch \
  --num_processes=1 --num_machines=1 --main_process_port=0 \
  --mixed_precision=bf16 --dynamo_backend=no \
  $(which lerobot-train) \
  --config_path=/root/workspace/VLA-RL/outputs/train/blocks_to_box_v2_pi05_full_bs2effective_30k/checkpoints/030000/pretrained_model/train_config.json \
  --resume=true \
  --batch_size=8 \
  --save_freq=10000 \
  --wandb.enable=false \
  --policy.push_to_hub=false \
  --steps=60000
```

# 部署
```bash
# 服务器
HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1 CUDA_VISIBLE_DEVICES=1 \
/root/miniforge/envs/evo-rl/bin/python -m lerobot.async_inference.policy_server \
  --host=127.0.0.1 \
  --port=8080 \
  --fps=30 \
  --inference_latency=0.2 \
  --obs_queue_timeout=2

# 本地
ssh -N -o ExitOnForwardFailure=yes \
  -L 8080:127.0.0.1:8080 \
  root@180.76.108.184

python -m lerobot.async_inference.robot_client \
  --robot.type=bi_so_follower \
  --robot.id=bi_so101_follower \
  --robot.calibration_dir=/media/aiden/Data/Dubuntu/lerobot_data/calibration/robots/so_follower \
  --robot.left_arm_config.port=/dev/ttyACM0 \
  --robot.right_arm_config.port=/dev/ttyACM1 \
  --robot.left_arm_config.max_relative_target=5.0 \
  --robot.right_arm_config.max_relative_target=5.0 \
  --robot.left_arm_config.cameras='{ wrist: {type: opencv, index_or_path: 2, width: 640, height: 480, fps: 30, fourcc: "MJPG"}}' \
  --robot.right_arm_config.cameras='{ wrist: {type: opencv, index_or_path: 4, width: 640, height: 480, fps: 30, fourcc: "MJPG"}, front: {type: opencv, index_or_path: 6, width: 640, height: 480, fps: 30, fourcc: "MJPG"}}' \
  --task="Pick up the objects and put them into the box." \
  --server_address=127.0.0.1:8080 \
  --policy_type=pi05 \
  --pretrained_name_or_path=/root/workspace/VLA-RL/outputs/train/blocks_to_box_v2_pi05_full_bs2effective_30k/checkpoints/030000/pretrained_model \
  --policy_device=cuda \
  --client_device=cpu \
  --actions_per_chunk=50 \
  --chunk_size_threshold=0.0 \
  --aggregate_fn_name=latest_only \
  --fps=30
```