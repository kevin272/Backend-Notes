# Backend-Fundamentals
These are some personal notes about backend used in MERN Stack application dev process. It can help you too if you are starting MERN Stack journey or have some speculations/confusions.

---

## Common Working Process Intro

### Server Setup
I generally setup a basic server to start the development process. A quick server.js/ index.js with app initialization via express and port env variable usage is preferred. I also predefine the CORS integration with a basic template frontend to make future checkings easy. 
The basic code may look like:
```js
import express from "express";
import dotenv from "dotenv";
import cors from "cors";
dotenv.config(); //Never forget this

const PORT = process.env.PORT || 5000;// I always use this port idk why
const app = express();

app.use(cors({origin:"http://localhost:...", credentials: true}));
app.use(express.json());//This parses response json in req.body

app.get("/",(req,res) => {
    res.send("Hello World!");
})

app.listen(PORT, () => {
    console.log("Server Running on http://localhost:"+ PORT)
})
```


### Folder/Directory Structure
THe most practical and efficient directory structure I have used till now is: 
```bash
src/
│
├── config/             # Database and app configuration
│
├── controllers/        # Business logic (handlers for routes)
│
├── models/             # Mongoose schemas
│
├── routes/             # Express route definitions
│
├── middleware/         # Custom middleware (e.g. auth, error handling)
│
├── utils/     # Helper functions 
│
├── validations/        # Joi/Yup/Zod schema validators 
```
outside of the src folder, the basic git files and package files with server.js is located

### Routes and Database Setup
A good way to start is to preplan what routes and db model we're going to use. I generally define these routes, test them and move forward. The general usecase is for authRouter/userRouter in basic projects and here are some useful tips: 
- You can add `"type": "module"` in package.json to use format of `"import ..from .."` instead of ``"const .. = require .."`
- Make sure to import correct types from files. You can mistakely import wrongly by not encloding the imports in **{}** brackets 
- For routes with specific id or token needed, you can use routes like this `router.post("/reset-password/:token",resetPassword)`

For schema creation, you generally use a **db.config.js** which uses the mongoose library. the connection code is always the same

```js
import mongoose from "mongoose";

export const connectDB = async () => {
	try {
		const conn = await mongoose.connect(process.env.MONGODB_URL);
		console.log(`MongoDB Connected: ${conn.connection.host}`);
	} catch (error) {
		console.log("Error connection to MongoDB: ", error.message);
		process.exit(1); // 1 is failure, 0 status code is success
	}
};
```
After this you can add `connectDB()`; to the server.

---

## Schema Creation and Common controller functions
Lets take a common example of userSchema and we want to signup,login and logout from the backend(Lets suppose routes have been defined). What we can do is use an auth.controller.js or user.controller.js(I prefer auth) that contains the controller functions. The need of separate file for this function is to make code cleaner and accessible for debugging.

### Schema
But first we want to create a userschema via user.model.js. **new mongoose.Schema({})** is used where we can pass the parameters. Example usecase:
```js
const userSchema = new mongoose.Schema({
          email:{
                    type: String,
                    required: true
          },
          password:{
                    type: String,
                    required: true
          }
},{timestamps:true})
```
The **timestamps:true** automatically creates 2 new fieilds called `createdAt` and `updatedAt` which help track records.
The Schema is just defined by the above code. The generation occurs by the line 
```js
export const User= mongoose.model('User',userSchema);
```
This creates a model named `User` based on the userSchema. This can be verified by checking via MongoDB compass but the collection must be already created.

****IMPORTANT NOTES/TIPS TO CONSIDER****

- Use `trim=true` for strings as it cleans the input. Eg: `" kebin " => "kebin"`
- For email, always use `match : /.+\@.+\..+/` and `unique:true`
- Use `enum` for roles/status. Eg: `role: { type: String, enum: ['admin', 'user'] }`

### Signup




