检查串口
ls /dev/ttyACM*

给串口权限
sudo chmod 666 /dev/ttyACM*

标定
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






遥操作：
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

# 双臂+摄像头
lerobot-teleoperate \
  --robot.type=bi_so_follower \
  --robot.id=bi_so101_follower \
  --robot.left_arm_config.port=/dev/ttyACM0 \
  --robot.right_arm_config.port=/dev/ttyACM1 \
  --robot.left_arm_config.cameras='{left: {type: opencv, index_or_path: 2, width: 640, height: 480, fps: 30, fourcc: "MJPG", backend: 200}}' \
  --robot.right_arm_config.cameras='{right: {type: opencv, index_or_path: 4, width: 640, height: 480, fps: 30, fourcc: "MJPG", backend: 200}, middle: {type: opencv, index_or_path: 6, width: 640, height: 480, fps: 30, fourcc: "MJPG", backend: 200}}' \
  --teleop.type=bi_so_leader \
  --teleop.id=bi_so101_leader \
  --teleop.left_arm_config.port=/dev/ttyACM2 \
  --teleop.right_arm_config.port=/dev/ttyACM3 \
  --display_data=true
```

录制数据
```bash
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
  --dataset.repo_id=aiden/blocks_to_box_test \
  --dataset.single_task="Pick up the objects and put them into the box." \
  --dataset.num_episodes=10 \
  --dataset.episode_time_s=100 \
  --dataset.reset_time_s=10 \
  --dataset.push_to_hub=false \
  --dataset.vcodec=h264 \
  --play_sounds=false \
  --display_data=true

```


查看数据
```bash
lerobot-dataset-report --dataset aiden/blocks_to_box_test

lerobot-dataset-viz \
  --repo-id aiden/blocks_to_box_test \
  --episode-index 0 \
  --display-compressed-images=false

```




训练
```bash
cd /root/workspace/VLA-RL

CUDA_VISIBLE_DEVICES=0,1 accelerate launch \
  --multi_gpu \
  --num_processes=2 \
  --mixed_precision=bf16 \
  $(which lerobot-train) \
  --dataset.repo_id=aiden/blocks_to_box_1 \
  --policy.type=pi05 \
  --policy.pretrained_path=/root/workspace/VLA-RL/checkpoints/lerobot/pi05_base \
  --policy.device=cuda \
  --policy.dtype=bfloat16 \
  --policy.gradient_checkpointing=true \
  --peft.method=LORA \
  --peft.r=16 \
  --batch_size=4 \
  --steps=30000 \
  --save_freq=10000 \
  --eval_freq=0 \
  --output_dir=/root/workspace/VLA-RL/outputs/train/blocks_to_box_pi05_lora_bs8effective_30k \
  --job_name=blocks_to_box_pi05_lora_bs8effective_30k \
  --wandb.enable=false \
  --policy.push_to_hub=false
```



部署
```bash
# 服务器
HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1 CUDA_VISIBLE_DEVICES=0 \
/root/miniforge/envs/evo-rl/bin/python -m lerobot.async_inference.policy_server \
  --host=127.0.0.1 \
  --port=8080 \
  --fps=5 \
  --inference_latency=0.2 \
  --obs_queue_timeout=2

# 本地
ssh -N -o ExitOnForwardFailure=yes \
  -L 8080:127.0.0.1:8080 \
  root@180.76.108.184


```