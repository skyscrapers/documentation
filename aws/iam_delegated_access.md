# How to provide Skyscrapers access to your AWS account

1. Create a new role. Choose type Another AWS account
2. Enter the ID of our NOC account 568346047405 and enable REQUIRE MFA
3. Attach the required policy (for example ReadOnlyAccess or AdministratorAccess)
4. Set Role name skyscrapers_full_operators
5. (I assume its still 114769043951?)
