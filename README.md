import json
import unittest
from unittest.mock import patch, Mock

import boto3

import lambda_function  # Your Lambda function implementation

class TestLambdaFunction(unittest.TestCase):
    @patch('boto3.resource')
    def test_delete_sqs_message(self, mock_resource):
        # Mock the SQS resource with patch
        mock_sqs = Mock()
        mock_resource.return_value = mock_sqs

        # Define the SQS message to delete
        receipt_handle = '1234'
        message_id = '5678'
        event = {
            'Records': [
                {
                    'receiptHandle': receipt_handle,
                    'messageId': message_id
                }
            ]
        }

        # Define the expected response from the SQS resource
        delete_response = {
            'ResponseMetadata': {
                'RequestId': '1111',
                'HTTPStatusCode': 200,
            }
        }

        # Configure the mock to return the expected response
        queue_name = '<your_queue_name>'
        queue_url = 'https://sqs.us-east-1.amazonaws.com/<your_account_id>/' + queue_name
        mock_queue = Mock()
        mock_queue.delete_messages.return_value = delete_response
        mock_sqs.Queue.return_value = mock_queue

        # Invoke the Lambda function with the mocked event and SQS resource
        response = lambda_function.lambda_handler(event, {})

        # Verify that the message was deleted and the expected response was returned
        expected_response = {
            'statusCode': 200,
            'body': json.dumps('Message deleted from queue')
        }
        self.assertEqual(response, expected_response)
