# Sequelize Model Association 

### Models can be defined two ways:
  * Calling sequelize.define(modelName, attributes, options)
  * Extending Model and calling init(attributes, options)

Both seem to be functionally equal (sequelize.define actually calls Model.init) however I prefered extending the Model, it felt like a more intuitive and concise way to define the model. *Upon extra research it seems extending the model is the newer method so this will probably have more longevity for support. The following example will reflect the latter definition.*

 I would suggest using the Sequelize CLI to create your models. Here is an example of creating a User and a Course model. When you generate the models using Sequelize CLI, the corresponding migration file is also created.

```$ npx sequelize-cli model:generate --name User --attributes firstName:string, lastName:string, emailAddresss:string, password:string```

```$ npx sequelize-cli model:generate --name Course --attributes id:integer, title:string, description:string```

Navigate to the User model, it should look like this after providing the necessary information. 

```
'use strict';
  const {
    Model
  } = require('sequelize');
  module.exports = (sequelize, DataTypes) => {
    class User extends Model {
      /**
      * Helper method for defining associations.
      * This method is not a part of Sequelize lifecycle.
      * The `models/index` file will call this method automatically.
      */
      static associate(models) {
        // define association here
      }
    };
    //Other model options go here
    User.init({
      firstName: DataTypes.STRING,
      lastName: DataTypes.STRING,
      emailAddress: DataTypes.STRING,
      password: DataTypes.STRING,
    }, {
      sequelize,
      modelName: 'User',
    });
    return User;
  };
```
For this specific project I had written and tested my models before defining the associations between them so the associations will be defined in the options parameter instead of before the attributes like the generated model suggests. Currently the model looks like this before defining the associations.

```
'use strict';
  const {
    Model
  } = require('sequelize');
  module.exports = (sequelize, DataTypes) => {
    class User extends Model {}
    };
    //Other model options go here
    User.init({
      firstName: DataTypes.STRING,
      lastName: DataTypes.STRING,
      emailAddress: DataTypes.STRING,
      password: DataTypes.STRING,
    }, {
      sequelize,
      modelName: 'User',
    });
    //association will go here
    return User;
  };
```
Associations connect two models by a single foreign key, so we need to edit **both** our models to reflect this relationship.
*****
###### There are other types of associations other than what is focused on for this project, if you would like to explore them click here: <https://sequelize.org/master/manual/assocs.html>
*****

**For this project, we are using a One-To-Many association.**

One-To-Many associations connect one source with multiple targets, while all these targets are connected with only this single source. 

##### Project specific explanation: We are trying to associate a single User to many Courses and a single Course to a single User. 

The relationship must be a two-way street, and while there are always exceptions, you will almost always see associations in pairs like this. The key words for this association are `hasMany()` and `belongsTo()`.

To tell Sequelize that you want an association, first a function must be called. Here is the structure of the function:

```
User.associate = function(models) {
  //where the association is defined
  Model.hasMany(models.targetModel, {
    //options
  })
};
```
Notice the `associate()` method receives a parameter of models, this contains every declared model within the models directory.
The `associate()` method is called in the db/index.js file after each model is imported into the Sequelize instance. This allows code within the `associate()` method to access *_ANY_* of the available models. (reference: Treehouse Sequelize Model Association Course).

**Now we can fill in our information to create our association.** 

To define a single User to many Courses, call the User model's `hasMany()` method, passing in a reference to the Course model:

```
User. associate = function(models) {
  User.hasMany(models.Course, {
    //options
  });
};
```

This tells Sequelize that a User can be associated with one or more(or "many") Courses. The Courses table will now contain a `UserId` foreign Key column. *This will be explained in more detail later when we are customizing the foreign key, however take note of it being capitalized*

For this example, a Course can only have ONE User so we use a One-to-One Association. A single course to a single user. This association includes the Course model's `belongsTo()` method passing in a reference to the User model:

```
Course.associate = function(models) {
  Course.belongsTo(models.User, {
      //options
  })
};
```

This tells Sequelize a Course can only be associated with only user.

*****
### Foreign Key

At this point if you run `npm start` or try to deploy your database on a website like Heroku, you will most likey receive an error related to a foreign key constraint. This is because the UserId column for the foreign key in the Course table(mentioned above) doesn't match the column naming convention of the other table columns. Sequelize automatilly defines the foreign key names. Sometimes this works for the program you've already written, sometimes it doesn't. In my case, it didn't because the generated column name is 'UserId', however when I run the database, it is searching for 'userId'. It is simple to specify a custom foreign key and in the long run may save you some time troubleshooting a bug in your associations.

**Let's set the foreign key name in both models:**

In order to do this we can finally write some code into the second argument of the `belongsTo()` and `hasMany()` methods. If we pass an options object here we can use the `foreignKey` property to specify the foreign key name like so:

models/course.js:

```
Course.associate = function(models) {
  Course.belongsTo(models.User, {
      foreignKey: 'userId'
  });
};
```

models/user.js:

```
User.associate = function(models) {
  User.hasMany(models.Course, {
      foreignKey: 'userId'
  });
};
```
*****
### Primary Key

Sequelize will automatically define a `primaryKey`, it uses `id` by default. However, this will not match with what our associations are defining so we must instruct Sequelize to generate the `primaryKey` column using the property name defined in the model (`foreignKey: 'userId'`). `userId` refers to the `id` column in the `Users` table. (This is important to remember, there is no `userId` column in the user table.) This is accomplished by setting `primaryKey` to true where the tables are being created. Navigate to the /migrations folder and find the file that ends with `-create-course.js`. This would have been generated when you created the Course model. Set `primaryKey: true`:

/migrations/....-create-course.js

```
wait queryInterface.createTable('Courses', {
  id: {
    allowNull: false,
    autoIncrement: true,
    primaryKey: true,
    type: Sequelize.INTEGER
  }
}
```
/migrations/....-create-user.js
```
await queryInterface.createTable('Users', {
  id: {
    allowNull: false,
    autoIncrement: true,
    primaryKey: true,
    type: Sequelize.INTEGER
  }
}
```
*****
### Migrations

One more change is necesasry to fully set up the relationship in the databse. Navigate back to the `create-course.js` file in /migrations. In this file we need to change the object labeled userId so that it references the correct information. 
*****
###### This was a place where I personally got held up. I didn't even have a userId object in this file so when I was first testing my database I was receiving "column UserId does not exist in 'Courses'". Sounds straight forward, but it took me a while to troubleshoot and to understand how all the moving parts of the association work together.
*****
Here is the full 'create-course.js' file in the /migrations folder:

```
'use strict';
  module.exports = {
    up: async (queryInterface, Sequelize) => {
      await queryInterface.createTable('Courses', {
        id: {
          allowNull: false,
          autoIncrement: true,
          primaryKey: true,
          type: Sequelize.INTEGER
        },
        title: {
          type: Sequelize.STRING
        },
        userId: {
          type: Sequelize.INTEGER,
          onDelete: 'CASCADE',
          references: {
            model: 'Users',
            key: 'id',
            as: 'userId',
          }
        },
        description: {
          type: Sequelize.TEXT
        },
        estimatedTime: {
          type: Sequelize.STRING
        },
        materialsNeeded: {
          type: Sequelize.STRING
        },
        createdAt: {
          allowNull: false,
          type: Sequelize.DATE
        },
        updatedAt: {
          allowNull: false,
          type: Sequelize.DATE
        }
      }
    );
  },
  down: async (queryInterface, Sequelize) => {
    await queryInterface.dropTable('Courses');
  }
};
```
  * `onDelete: 'CASCADE'` configures the model so if a user is deleted, the user's Course will be deleted as well.
  * `references` section will set up the 'Courses' table in the database to reflect the same relationship we set up in the inital steps of this.

*****
### Defining an Alias

At this point, you could have everything working just great however I was still receiving an error `column 'UserId' does not exist`. For me, this was because of how I was requesting information in my GET request. If you're still receiving this error, navigate to your routes.js file and check out your GET request.

Mine looks like this: 

```
router.get('/courses', async (req, res) => {
  try {
  const courses = await Course.findAll({
      include: {
          model: User,
          as: 'user',
          attributes: ["id", "firstName", "lastName", "emailAddress" ]
      },
      attributes: ["id", "title", "description", "estimatedTime", "materialsNeeded"]
    });
  console.log(Course);
  res.json(courses);
  res.status(200).end();
  } catch (error) {
    res.status(500).json({ message: error.message });
    }
  }
);
```
Notice how I've used the include property, this allows us to indicate specific information we want with every Course. In this case, we want to include User information for each Course. "Include the User model as 'user' with these attributes.". If I commented out the whole 'include' property and deployed my database everything worked, except for the obvious problem I was missing the User information. But at least this helped to narrow down where the issue was.

In the include property I've defined an alias with `as: 'user'`. Because of this, when the data is returned `User` is changed to `user`. This needs to be reflected in *BOTH* models. Creating an alias for the model association is as simple as adding an `as` property to the `belongsTo()` and `hasMany()` method options object literal like so:

models/course.js:

```
Course.associate = function(models) {
  Course.belongsTo(models.User, {
      foreignKey: 'userId',
      as: 'user'
  });
};
```

models/user.js:

```
User.associate = function(models) {
  User.hasMany(models.Course, {
      foreignKey: 'userId',
      as: 'user'
  });
};
```
*****
###### It is safest practice to specify the foreign key if you are going to specify an alias. Simply using `as` to change the name of the association will also change the name of the foreign key.
*****

Now, everything should be connected. Our models are calling the `associate()` method which connects one User to many Courses as well as one Course to one User. This association's foreign key is `userId` and this is all contained in a user object with the alias of `user`. 


If you're still running into problems, almost all of my problems I encountered were because of the userId object in the 'create-course.js' file in the /migration folder. Painstakingly double checking your capitals vs. lower cases is a good place to start. It is easy to get them confused before you even realize it. 
