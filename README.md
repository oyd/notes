# notes

import os
from unittest.mock import MagicMock, patch

import boto3
from moto import mock_aws

import monthly_ec2_refresh as mod


@mock_aws
def test_lambda_terminates_instance_and_publishes_sns():
    region = "us-east-1"
    instance_name = "ew-data-ingestion-instance"

    # Create a running instance + Name tag (in moto EC2)
    ec2 = boto3.client("ec2", region_name=region)
    resp = ec2.run_instances(ImageId="ami-12345678", MinCount=1, MaxCount=1)
    instance_id = resp["Instances"][0]["InstanceId"]
    ec2.create_tags(Resources=[instance_id], Tags=[{"Key": "Name", "Value": instance_name}])

    # Env vars your lambda expects
    os.environ["REGION"] = region
    os.environ["SNS_ARN"] = "arn:aws:sns:us-east-1:123456789012:dummy-topic"
    os.environ["INSTANCE_NAME"] = instance_name

    # Patch boto3.client only for SNS, keep real moto EC2 client for everything else
    real_boto3_client = boto3.client
    sns_mock = MagicMock()

    def client_side_effect(service_name, *args, **kwargs):
        if service_name == "sns":
            return sns_mock
        return real_boto3_client(service_name, *args, **kwargs)

    with patch("boto3.client", side_effect=client_side_effect):
        result = mod.lambda_handler({}, {})

    assert result["terminated"] == 1
    assert result["instance_ids"] == [instance_id]

    # Instance should no longer be running
    state = ec2.describe_instances(InstanceIds=[instance_id])["Reservations"][0]["Instances"][0]["State"]["Name"]
    assert state in {"shutting-down", "terminated"}

    # SNS publish should have been called
    assert sns_mock.publish.call_count == 1
