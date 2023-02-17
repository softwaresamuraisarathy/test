import json
import boto3

def lambda_handler(event, context):
    # Get the SQS message from the event
    message = event['Records'][0]
    receipt_handle = message['receiptHandle']
    queue_name = '<your_queue_name>'
    region_name = 'us-east-1'

    # Delete the message from the SQS queue
    sqs_resource = boto3.resource('sqs', region_name=region_name)
    queue = sqs_resource.get_queue_by_name(QueueName=queue_name)
    queue.delete_messages(Entries=[{'Id': message['messageId'], 'ReceiptHandle': receipt_handle}])

    # Do other processing with the SQS message
    # ...

    return {
        'statusCode': 200,
        'body': json.dumps('Message deleted from queue')
    }
