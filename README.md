# Backend-Fundamentals
These are some personal notes about backend used in MERN Stack application dev process.  I used it as my interview prep session and it can help you too if you are starting MERN Stack journey or have some speculations/confusions.

---

## Common Working Process Intro

### Server Setup
I generally setup a basic server to start the development process. A quick server.js/ index.js with app initialization via express and port env variable usage is preferred. I also predefine the CORS integration with a basic template frontend to make future checkings easy. 
The basic code may look like:
```js
import express from "express";
import dotenv from "dotenv";
import cors from "cors";
import cookieParser from "cookie-parser";

dotenv.config(); //Never forget this

const PORT = process.env.PORT || 5000;// I always use this port idk why
const app = express();

app.use(cors({origin:"http://localhost:...", credentials: true}));
app.use(express.json());//This parses response json in req.body
app.use(cookieParser()); //I will explain later but in general token and cookies usage


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
For Signup, there are different technologies and functions that we need to consider.

##### Bcrypt
When the user enters the password via signup, the password is directly stored to the database. During databreaches, the attackers can directly access the password which is not preferred. `Bcrypt` is a password hashing library that can overcome it by making it non-readable and irreversible.
A simple hash can be generated by using `bcryptjs.hash(password,10)`. 

### JWT and Cookies
This is the main auth component. Token is used for **verification** or ****authorization**** and cookies are for local storage of the token.
For the token generation to work, a JWT_SECRET env variable needs to be set with is generally a set of constant random characters that we must set. Token can generated by `jwt.sign({}).JWTSECRET` and it's expiration time can also be set seen in this example code: 
```js
export const generateTokenAndSetCookie = (res, userId) => {
	const token = jwt.sign({ userId }, process.env.JWT_SECRET, {
		expiresIn: "7d",
	});

	res.cookie("token", token, {
		httpOnly: true,
		secure: process.env.NODE_ENV === "production",
		sameSite: "strict",
		maxAge: 7 * 24 * 60 * 60 * 1000,
	});

	return token;
};
```
In general, the logic is that when tokengeneration function is called, userID is also passed which is used to sign the JWT token. During creation, a cookie is also generated which lasts for 7 days and stores the token as "token".

Also, if there is need of verification code for users to complete signup, this code can be used: 
```js
export const generateVerificationCode = () => {
    return Math.floor(100000 + Math.random() * 900000).toString();
}
```
I generally place these functions in the `helper.js`

The generated verificationToken and verificationCode can be used to send verificationEmail to user via mailtrap SMTP.
#### MailTrap
It is a very simple utility toolkit for sending mails from the server. We can use custom templates or use from the templates provided. Generally, nodemailer is used for simple mails such as verification emails and welcome emails.

Firstly a config is needed to setup mailtrap which needs MAILTRAP_TOKEN that is obtained from creating an account from their website.
```js
import { MailtrapTransport } from "mailtrap";
import dotenv from "dotenv";
import Nodemailer from "nodemailer";
dotenv.config();

export const mailtrapClient = Nodemailer.createTransport(
  MailtrapTransport({
    token: process.env.MAILTRAP_TOKEN,
  })
);

export const sender = {
  address: "hello@gmail.com.np",
  name: "Company_name",
};
```

The structure of mail sending is in the form of `from:, to: , subject: , html: ,category: ,`.
You might be confused by the html part but it is the section of using template for the mail. We pass the `email` and `verificationToken` to the email sending function and replace the placeholder value inside the template with the parameter value.
```js
export const sendVerificationEmail =  async (email,verificationToken) => {
    const recipient = [email];

    try {
        const response = await mailtrapClient.sendMail({
            from: sender,
            to: recipient,
            subject: "Verify your email",
            html: VERIFICATION_EMAIL_TEMPLATE.replace("verificationCode",verificationToken),
            category: "Email Verification"
        })

        console.log("Email sent successfully",response);
        
    } catch (error) {
        console.error(`Error sending verification`,error);
        throw new Error(`Error sending verification email: ${error}`)
    }
}
```

The full function block for signup controller is: 

```js
export const signup = async (req,res) => {
    const{email,password,name} = req.body;
    try {
        if(!email||!password||!name){
            throw new Error("All fields are required");
        }
        const userAlreadyExists = await User.findOne({email});
        if (userAlreadyExists){return res.status(400).json({success:false, message: "User Already Exists"})};
        
    } catch (error) {
        return res.status(400).json({success:false, message: error.message});
        
    }
    const hashedPassword = await bcryptjs.hash(password,10);
    const verificationToken  = generateVerificationCode();
    
    const user = new User({  //Create new user Document
        email,
        password: hashedPassword,
        name,
        verificationToken,
        verificationTokenExpiresAt: Date.now() + 24 * 60 * 60 *1000 //24hrs
    })

    await user.save(); // Save the user to the Database

    generateTokenAndSetCookie(res,user._id);
    
    await sendVerificationEmail(user.email, verificationToken);

    res.status(201).json({
        success: true,
        message: "User created successfully",
        user: {
            ...user._doc,
            password: undefined,
        },
    })
}
```

If we understand the signup process, login and logout become wayy to easy.

### Login
In the login function, email and password are passed as arguments. We can first check if user exits or not by `User.findOne({email})`. After that, we can use `bcrypt.compare` to compare between input `password` and `User.password`. The output is true or false. If true, call `GenerateTokenandSetCookie` and return response with status.

```js
export const login = async (req,res) => {
    const {email,password} = req.body;
    try {
        const user = await User.findOne({email});
        if(!user){return res.status(400).json({success:false, message: "Invalid credentials"})
        };
    const isPasswordValid = await bcryptjs.compare(password, user.password);
    if(!isPasswordValid){return res.status(400).json({success:false, message: "Invalid credentials"})}
    generateTokenAndSetCookie(res,user._id);
    user.lastLogin= new Date();
    await user.save();
    
    res.status(200).json({
        success:true,
        message:"Loggedin Successfully",
        user: {
            ...user._doc,
            password: undefined
        }
    })
    } 
    catch (error) {
        console.log("Error in Login",error);
        res.status(400).json({success:false, message: error.message});
        
    } 
}
```

### Logout
This is very simple. When accessed the function, clear the cookies and send status code.
```js
export const logout = async (req,res) => {
    res.clearCookie("token");
    res.status(200).json({
        success:true,
        message: "Logged Out successfully"
    });
}
```
****IMPORTANT NOTES/TIPS TO CONSIDER****
- Use `try/catch` in async functions.
- Use `Postman` API calls to verify each endpoint
- Use `.env` for all secrets 
- Never expose internal error details to the client
- Use `JWT tokens` + `httpOnly` cookies for secure session auth.


