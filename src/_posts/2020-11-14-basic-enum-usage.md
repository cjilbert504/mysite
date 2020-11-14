---
layout: post
title:  "Basic Enum Usage for Defining Roles or Statuses on Models in Rails"
date:   2020-11-14 14:52:31 -0600
categories: rails
---

Have you ever wanted to have a number of different options for the value of an attribute, say on a User model, such as an attribute called `role` that could be any of the following values: 

admin, volunteer, customer, etc. 

Perhaps you're working on a small version of Airbnb and you want the rental properties to have an availability status but you don't want it based in a boolean type attribute. There are plenty of gems that you could use to achieve this goal but let's explore something that comes with Rails out of the box.

Enter [ActiveRecord::Enum](https://api.rubyonrails.org/v5.2.3/classes/ActiveRecord/Enum.html). From the api docs: 

> Declare an enum attribute where the values map to integers in the database, but can be queried by name.

What this means is that you would define a column on the database table for the model that you are wanting to add roles or status options to that would be of type integer. Inside the model file for the model you would declare an enum and pass the options that you would like to have for the roles or statuses as an array of symbols.

***NOTE - you can also explicitly map the attribute and the integer using a hash instead if desired.***

The integers from the role column on the users database table are mapped to the index's of the array holding the attribute options. 

The other nice thing here is that Enum provides you with a handful of dynamically defined methods for querying and updating an instances value for the attribute.

Let's generate a new rails app and play around with this feature to practice and see what kind of fun things we get for free.
```ruby
rails new enum_practice
cd enum_practice/
```
Now let's generate a quick model and migration for a User model. 
```ruby
rails g model User name role:integer --no-test-framework
```
The above command will generate a `user.rb` file and a migration file to create the users table with the columns: `name` - which is of type string (the default type if no type is specified)
`role` - This will be the enum column and again it's of type integer

Let's look at the migration file we got from running the above command:
```ruby
class CreateUsers < ActiveRecord::Migration[6.0]
  def change
    create_table :users do |t|
      t.string :name
      t.integer :role

      t.timestamps
    end
  end
end
```

If we want to have a default role set when a new user gets created we can pass that default option on the `t.integer :role` line. We will do this but first we need to decide what our initial list of roles will be for this model.

Let's say there are two roles we want on the user model for now: admin and volunteer.
To do this add this line to your user.rb file:
```ruby
class User < ApplicationRecord
	enum role: [:volunteer, :admin]
end
```
Alternatively, you can write that line in a syntax that you will probably see more often out in the wild:
```ruby
class User < ApplicationRecord
	enum role: %i(volunteer admin)
end
```
Both create an array of symbols.

We probably don't want every new user that gets created to be an admin so let's setup our migration so that the default role of all new users is `:volunteer`.

Looking at the migration file again, we are reminded that the column we want to set a default on is of type integer so our default value needs to be just that, an integer. What integer value do we want here? Well looking at our array of roles again we want `:volunteer` which is at index 0 of the `enum role` array. Let's add the line needed to set this default.
```ruby
class CreateUsers < ActiveRecord::Migration[6.0]
  def change
    create_table :users do |t|
      t.string :name
      t.integer :role, default: 0

      t.timestamps
    end
  end
end
```
Let's run the migration and then hop into the rails console:
```ruby
bin/rails db:migrate
bin/rails c
```
Ok, new user time!
```ruby
user = User.new(name: "Collin")
 => #<User id: nil, name: "Collin", role: "volunteer", created_at: nil, updated_at: nil> 
```
Looks like it's working! We only passed in a value for name when instantiating a new User object but since we setup a default for role that default value is assigned to our new instances role attribute. 

Let's now explore some of the built in methods we get from using Enum.

We can send the object a message to find out what role it has:
```ruby
user.role
 => "volunteer" 
```
We can send the object a message to get a boolean value back based on whether or not the user is a volunteer:
```ruby
user.volunteer?
 => true 
```
Similarly we can send the object a message to find out if it is an admin:
```ruby
user.admin?
 => false 
```
 
This user has been a big help to the organization and deserves a promotion to admin status! Easy enough to do:
```ruby
 user.admin!
   (0.1ms)  begin transaction
  User Create (0.7ms)  INSERT INTO "users" ("name", "role", "created_at", "updated_at") VALUES (?, ?, ?, ?)  [["name", "Collin"], ["role", 1], ["created_at", "2020-11-11 17:09:53.227728"], ["updated_at", "2020-11-11 17:09:53.227728"]]
   (1.0ms)  commit transaction
 => true 
```
Here we can see the SQL that was fired off, notice the role has been changed to reflect the integer 1. Did this actually update the role of our user? Let's send a message and find out!
```ruby
user.admin?
 => true
```
Other ways to change the role for the user are:
```ruby
user.role = "volunteer"
 => "volunteer" 
user.role
 => "volunteer"
```
and
```ruby
user.role = :admin
 => :admin 
user.role
 => "admin"
```
**An important thing to note with the above two ways though is that you will have to call save after updating the role in order to persist the change**

Now let's take a look at some query methods for getting either all admins or all volunteers. First let's create four new User instances:
```ruby
names = %w(Uma Jordan Henry Collin)
names.each {|name| User.create(name: name)}
```
Now let's change the first two users to be admins:
```ruby
two_users = User.limit(2)
two_users.each {|user| user.admin!}
```
Now we have two admin users and two volunteer users in the database. Let's get all the admin users:
```ruby
User.admin
  User Load (0.4ms)  SELECT "users".* FROM "users" WHERE "users"."role" = ? LIMIT ?  [["role", 1], ["LIMIT", 11]]
 => #<ActiveRecord::Relation [#<User id: 2, name: "Uma", role: "admin", created_at: "2020-11-11 17:17:46", updated_at: "2020-11-11 17:25:50">, #<User id: 3, name: "Jordan", role: "admin", created_at: "2020-11-11 17:17:46", updated_at: "2020-11-11 17:25:50">]>
```
What about all the volunteers?
```ruby
User.volunteer
  User Load (0.4ms)  SELECT "users".* FROM "users" WHERE "users"."role" = ? LIMIT ?  [["role", 0], ["LIMIT", 11]]
 => #<ActiveRecord::Relation [#<User id: 4, name: "Henry", role: "volunteer", created_at: "2020-11-11 17:17:46", updated_at: "2020-11-11 17:17:46">, #<User id: 5, name: "Collin", role: "volunteer", created_at: "2020-11-11 17:17:46", updated_at: "2020-11-11 17:17:46">]>
```
So by calling one of the role options on the User class itself we can get back an array of all the instances that match the role message we send to User. Pretty cool!

We can also quickly send another message to the User class to find out what all the role options are and the corresponding integer values with:
```ruby
User.roles
 => {"volunteer"=>0, "admin"=>1}
```
Simply pluralize the name of the column and send that message to the class. We named the column `role` so we simply pluralized it( `roles` ) and sent that message to the `User` class (`User.roles`) and got back the hash of options and their integer values. Nifty indeed!

As a final note, using enum makes it incredibly easy to define new options on the model. Simply add the new option(s) to the end of the `enum role: [:volunteer, :admin]` array and that's it! The new options will be mapped to the next integer values.
Example:
```ruby
class User < ApplicationRecord
	enum role: [:volunteer, :admin, :vendor, :customer]
end

User.roles
 => {"volunteer"=>0, "admin"=>1, "vendor"=>2, "customer"=>3}
```
or in the alternate syntax style:
```ruby
class User < ApplicationRecord
	enum role: %i(volunteer admin vendor customer)
end

User.roles
 => {"volunteer"=>0, "admin"=>1, "vendor"=>2, "customer"=>3}
```


Happy Rubying!