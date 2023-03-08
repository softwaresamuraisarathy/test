
   
import unittest
import pandas as pd
from moto import mock_s3
import boto3


class TestS3CSVRead(unittest.TestCase):

    @mock_s3
    def test_read_csv_files(self):
        # Set up the mock S3 bucket and CSV files
        conn = boto3.resource('s3', region_name='us-east-1')
        bucket_name = 'test-bucket'
        conn.create_bucket(Bucket=bucket_name)
        csv_data = 'A,B\n1,2\n3,4\n'
        conn.Object(bucket_name, 'folder1/file1.csv').put(Body=csv_data)
        conn.Object(bucket_name, 'folder1/file2.csv').put(Body=csv_data)
        
        # Create an S3 resource
        s3 = boto3.resource('s3')
        
        # Specify the S3 bucket and folder
        prefix = 'folder1/'
        
        # Use the bucket and prefix to create a list of objects in the folder
        objects = s3.Bucket(bucket_name).objects.filter(Prefix=prefix)
        
        # Create an empty list to store the dataframes
        dfs = []
        
        # Loop through the objects and read each CSV file into a dataframe
        for obj in objects:
            # Get the object key (i.e., file name)
            key = obj.key
            
            # Read the CSV file into a dataframe
            df = pd.read_csv(f's3://{bucket_name}/{key}')
            
            # Append the dataframe to the list
            dfs.append(df)
        
        # Concatenate all the dataframes into one
        df = pd.concat(dfs, ignore_index=True)
        
        # Assert that the concatenated dataframe has the expected shape and values
        self.assertEqual(df.shape, (4, 2))
        self.assertTrue(df['A'].equals(pd.Series([1, 3, 1, 3])))
        self.assertTrue(df['B'].equals(pd.Series([2, 4, 2, 4])))

if __name__ == '__main__':
    unittest.main()
