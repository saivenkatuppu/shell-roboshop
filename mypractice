#!/bin/bash

# === CONFIGURATION ===
AMI_ID="ami-09c813fb71547fc4f"
INSTANCE_TYPE="t3.micro"
HOSTED_ZONE_ID="Z04121283NNLWE4N00ISG"
DOMAIN_SUFFIX="calvio.store"
REGION="us-east-1"
SECURITY_GROUP_ID="sg-0d8eafa2d644955f0"

# === Instance Names ===
SERVICES=("MongoDB" "Catalogue" "Redis" "User" "Cart" "MySQL" "Shipping" "RabbitMQ" "Payment" "Dispatch" "Frontend")

echo "Starting deployment..."

# === Instance tracking ===
declare -A INSTANCE_IDS
declare -A PRIVATE_IPS
declare -A PUBLIC_IPS

# === Function to get existing instance by name ===
get_instance_id_by_name() {
  local NAME=$1
  aws ec2 describe-instances \
    --region "$REGION" \
    --filters "Name=tag:Name,Values=$NAME" "Name=instance-state-name,Values=running,stopped" \
    --query 'Reservations[].Instances[].InstanceId' \
    --output text
}

# === Create or reuse instances ===
for SERVICE in "${SERVICES[@]}"; do
  echo "Checking instance for $SERVICE..."
  EXISTING_ID=$(get_instance_id_by_name "$SERVICE")

  if [ -n "$EXISTING_ID" ]; then
    echo "$SERVICE already exists: $EXISTING_ID"
    INSTANCE_IDS["$SERVICE"]=$EXISTING_ID
  else
    echo "Launching new instance for $SERVICE..."
    INSTANCE_ID=$(aws ec2 run-instances \
      --image-id "$AMI_ID" \
      --instance-type "$INSTANCE_TYPE" \
      --security-group-ids "$SECURITY_GROUP_ID" \
      --region "$REGION" \
      --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$SERVICE}]" \
      --query 'Instances[0].InstanceId' --output text)

    if [ -z "$INSTANCE_ID" ]; then
      echo "Failed to launch $SERVICE"
      continue
    fi

    INSTANCE_IDS["$SERVICE"]=$INSTANCE_ID
    echo "$SERVICE - Instance ID: $INSTANCE_ID"
  fi
done

# === Wait for all instances to run ===
echo "Waiting for instances to be in 'running' state..."
aws ec2 wait instance-running --instance-ids "${INSTANCE_IDS[@]}" --region "$REGION"

# === Get IPs ===
for SERVICE in "${SERVICES[@]}"; do
  INSTANCE_ID=${INSTANCE_IDS["$SERVICE"]}

  PRIVATE_IP=$(aws ec2 describe-instances \
    --instance-ids "$INSTANCE_ID" \
    --region "$REGION" \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' \
    --output text)

  PRIVATE_IPS["$SERVICE"]=$PRIVATE_IP

  if [ "$SERVICE" == "Frontend" ]; then
    PUBLIC_IP=$(aws ec2 describe-instances \
      --instance-ids "$INSTANCE_ID" \
      --region "$REGION" \
      --query 'Reservations[0].Instances[0].PublicIpAddress' \
      --output text)
    PUBLIC_IPS["$SERVICE"]=$PUBLIC_IP
  fi

  echo "$SERVICE - Private IP: $PRIVATE_IP, Public IP: ${PUBLIC_IPS[$SERVICE]}"
done

# === Create Route 53 Records ===
echo "Creating Route 53 A records..."

for SERVICE in "${SERVICES[@]}"; do
  RECORD_NAME="${SERVICE,,}.$DOMAIN_SUFFIX"  # lowercase
  if [ "$SERVICE" == "Frontend" ]; then
    IP=${PUBLIC_IPS["$SERVICE"]}
  else
    IP=${PRIVATE_IPS["$SERVICE"]}
  fi

  cat > change-${SERVICE}.json <<EOF
{
  "Comment": "Route 53 record for $SERVICE",
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "$RECORD_NAME",
        "Type": "A",
        "TTL": 1,
        "ResourceRecords": [{ "Value": "$IP" }]
      }
    }
  ]
}
EOF

  aws route53 change-resource-record-sets \
    --hosted-zone-id "$HOSTED_ZONE_ID" \
    --region "$REGION" \
    --change-batch file://change-${SERVICE}.json

  rm -f change-${SERVICE}.json
  echo "DNS record created: $RECORD_NAME -> $IP"
done

echo "Deployment completed."