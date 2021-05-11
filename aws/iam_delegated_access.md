# How to provide Skyscrapers access to your AWS account

1. Log into the AWS Console of the account you want to give us access to
2. Go to IAM
3. Create a new role. Choose type Another AWS account
4. Enter the ID of our NOC account `568346047405` and enable **REQUIRE MFA**
5. Attach the required policy (for example `ReadOnlyAccess` or `AdministratorAccess`)
6. Set Role name `skyscrapers_full_operators`
7. Provide us your account IDs
