# Launch instance in Public 1A
aws ec2 run-instances --image-id <value> --instance-type <value> --security-group-ids <value> --subnet-id <value> --key-name cloud-labs-nv --user-data <value>

aws ec2 run-instances --image-id ami-05548f9cecf47b442 --instance-type t2.micro --security-group-ids sg-0def0a95d2caa9be3 --subnet-id subnet-08f53ac471b6fa9a3 --key-name cloud-labs-nv --user-data file://user-data-subnet-id.txt

# Launch instance in Public 1B


# Launch instance in Private 1B


# Terminate instances

aws ec2 terminate-instances --instance-ids <value> <value>