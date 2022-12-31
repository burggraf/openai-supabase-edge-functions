# Summoning the Magical Powers of ChatGPT from your Supabase Edge Functions

As a software developer, I'm always looking for ways to make my applications more powerful and efficient. In this tutorial, I'll show you how I added OpenAI calls to my Supabase edge functions, bringing the magic of AI to my applications in just a couple of minutes. By incorporating calls to OpenAI's state-of-the-art language models, I can add natural language processing capabilities, generate text, translate between languages, or nearly anything else I can think of - all within a single edge function. 

What's more, it's integrated directly with my Supabase (PostgreSQL) database, so I can read existing data supplied by a user, automatically generate content, then store that content back in my database -- immediately when the data is entered, or on demand at any time.

### Seen any good movies lately?

Our sample app will collect a short list of movies that a user has enjoyed, and based on that set, we’ll recommend other movies they may also find interesting.  In short, we feed a list of movies to `OpenAI` and get back a list of recommendations.

We’ll use the `OpenAI Completions API` with the `text-davinci-003` model to get our movie recommendations from our Supabase Edge Function.  This core model, used extensively by ChatGPT, is perfect for this use case.

### Getting Started

#### Set up a Supabase Project

If you don’t already have a Supabase project, you’ll want to set one up at [supabase.com](https://supabase.com).  A project on the free tier will work just fine here.

#### Get an OpenAI API Key

You'll need an API key to make calls to OpenAI.  We'll use this key later when we set up our edge function.  For now, though, you'll just need to:

- Create an OpenAI account: [openai.com/api](https://openai.com/api/)
- Create an API Key:
    - Click on the icon at the top right
    - Select `View API keys` from the menu
    - Click `+ Create new secret key`
    - Copy your new key to the clipboard
    - Save it for use later (keep it stored in a secure location and do not share it with anyone)
As an example, your API key should look something like this:
`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`

#### Set up the Supabase CLI (command line interface)

Once your project is set up, you can follow the [Edge Functions Quickstart]( https://supabase.com/docs/guides/functions/quickstart) to get your function up and running.  Specifically, you’ll need to:
-	Install the Supabase CLI (command line interface). [see guide]( https://supabase.com/docs/guides/cli)
-	Login to the CLI using the command: `supabase login`. [see guide](https://supabase.com/docs/reference/cli/usage#supabase-login)
-	Initialize Supabase inside your project using the command: `supabase init`. [see guide](https://supabase.com/docs/guides/cli/local-development#getting-started)
-	Link to your Remote Project using the command `supabase link --project-ref your-project-ref`. [see guide](https://supabase.com/docs/reference/cli/usage#supabase-link)
-	Optional: Setup your environment: Follow [this setup guide](https://deno.land/manual/getting_started/setup_your_environment) to integrate the Deno language server with your editor.

#### Create the Function: `recommendations`

Following along in the Quickstart guide, you can create a function called `recommendations` to pick relevant movies.  Call it `recommendations`.  From inside your project folder, run:
`supabase functions new recommendations`

This will create a folder structure in your app that looks like this:
`supabase/functions/recommendations/`
and your function will be inside the file:
`supabase/functions/recommendations/index.ts`
Open that file in your text editor, and we’ll set it up to make the API call to OpenAI.

#### Editing the Function

The default function is a simple “hello, world” function that just takes a name as an input parameter and outputs a hello message with the name you passed it:

```js
import { serve } from "https://deno.land/std@0.168.0/http/server.ts"

console.log("Hello from Functions!")

serve(async (req) => {
  const { name } = await req.json()
  const data = {
    message: `Hello ${name}!`,
  }

  return new Response(
    JSON.stringify(data),
    { headers: { "Content-Type": "application/json" } },
  )
})
```

##### The `createCompletion` Utility Function

This is a great starting point, but we’ll want to modify it to call the OpenAI API.  Start by removing the `console.log("Hello from Functions!")` line.  In it's place, we'll create a new utility function that takes a `prompt` string and sends it to `OpenAI` to get a result.

```js
export const createCompletion = async (prompt: string) => {
  const response = await fetch(
      `https://api.openai.com/v1/engines/text-davinci-003/completions`,
      {
        body: JSON.stringify({
          prompt: prompt,
          temperature: 0.5,
          max_tokens: 512,
          top_p: 1,
          frequency_penalty: 0.52,
          presence_penalty: 0.5,
          best_of: 1
        }),
        headers: {
          Authorization: `Bearer ${Deno.env.get('OPENAI_API_KEY')}`,
          "Content-Type": "application/json",
        },
        method: "POST",
      },
    );
    return response.text();    
}
```

Let's look at this utility function and break it down:
- It's a basic Typescript async function.  (It's super important that this function works asynchronously, because we have no idea how long it's going to take OpenAI to calculate an answer for us.)
- It takes a single string parameter, which is the prompt we'll be sending to OpenAI.  This is the same type of prompt you'd use if you opened up ChatGPT in a web browser and typed out a prompt in the chat window.
- It uses the standard Javascript `fetch()` function to call the OpenAI API. 
- We're calling the `completions` API endpoint of the OpenAPI `text-davinci-003` model, located here: `https://api.openai.com/v1/engines/text-davinci-003/completions`
- Our `prompt` is sent in the `body` of the `fetch` call, along with some other parameters used by the `completions` API.  These options are documented here: [OpenAI: Create Completion](https://beta.openai.com/docs/api-reference/completions)
- We're sending our OpenAI API Key (that we created and saved earlier) in the `Authorization` header. (More on this below.)
- This is a `POST` call to the API (as opposed to a `GET` call).
- We return the `.text()` method from our function.  (When I tried returning `.json()`, the results would get truncated, so I found that returning `.text()` just worked more consistently.)

###### Managing the OpenAI API Key

Our OpenAI API Key is sent in the `Authorization` header using a Deno environment variable we're going to call `OPENAI_API_KEY` like this:

`Deno.env.get('OPENAI_API_KEY')`

We could just has well have hard-coded our key like this:

```js
headers: {
          Authorization: `Bearer <secret_openai_api_key>`,
          "Content-Type": "application/json",
        },
```

But that's a really bad security practice, and we never want to do that.  Plus, if our security key is compromised we'd have to come back and modify our source code and re-deploy our function.  

Luckily, there's a better way to handle secret keys like this with Supabase Edge Functions.  We just use the Supabase CLI `secrets` keyword to save our secret in a protected area on the server:

`supabase secrets set OPENAI_API_KEY=<secret_openai_api_key>`

Now that our secret is securely stored at the server, we can use it in our code by calling the built-in Deno function like this: `Deno.env.get('OPENAI_API_KEY')`.  If our key ever gets compromised, we just delete it from the OpenAI dashboard, generate a new one, and call:

```sh
supabase secrets unset OPENAI_API_KEY
supabase secrets set OPENAI_API_KEY=<new_secret_openai_key>
```

##### Editing the `serve` Function

Now, we'll need to edit the `serve` function that handles requests.  We'll need to:
- Take our list of `movies` sent as an array as input
- Massage that list and turn it into a nice-looking list of movies we can use in our prompt
- Build our text prompt into English that the `OpenAI Completion API` can understand
- Send our completed prompt to the `createCompletion` utility function
- Parse the response, extract the response text we want, and sent return it to our app 

Here is our edited `serve` function:

```js
serve(async (req) => {
  const { movies } = await req.json()
  let movieList = ""
  movies.forEach((movie: string) => {
    if (movieList.length > 0) movieList += ", "
    movieList += `"${movie.replace(/"/g, '\\"')}"`
  })
  const fullPrompt = `I enjoyed the following movies: ${movieList}. Can you suggest some other movies I might like?`
  const result = await createCompletion(fullPrompt)
  try {
    const jsonData: any = JSON.parse(result)
    const data = jsonData.choices[0].text
    return new Response(
      JSON.stringify({data, error: null}),
      { headers: { "Content-Type": "application/json" } },
    )      
  } catch (err) {
    return new Response(
      JSON.stringify({data: null, error: err}),
      { headers: { "Content-Type": "application/json" } },
    )
  }
})
```

Breaking it down:
- `movies` is passed to the function as a `json` object exactly the same way the `hello, world` function passed `name` to the function.  The only change here is that our parameter is now 1. called `movies` and 2. is now an `array` of `strings` (so we can pass several movies)
- We use a `forEach` loop to turn the array of movies into a string of movies that we can insert into our prompt
- If we send an array of movies: `["Raiders of the Lost Ark", "Back to the Future", "Die Hard"]` our prompt will end up looking like this: `I enjoyed the following movies: "Raiders of the Lost Ark", "Back to the Future", "Die Hard". Can you suggest some other movies I might like?`
- We send this prompt to `createCompletion` and wait for a `result`
- The result is parsed to a `JSON` object called `jsonData`
- We extract the `data` we want from `jsonData.choices[0].text` and return it back to our application

#### Deploy the `recommendations` Function

We should now be able to deploy our `recommendations` function and start calling it.  Using the `supabase cli` we just call:
`supabase functions deploy recommendations`

If we don't get any errors back, our function should be ready to use.  When we make changes to our function later, we'll need to call `supabase functions deploy recommendations` to deploy the new version each time.

### Testing

Let's test our function from the command line using `curl`.  If we go into the Functions section our Supabase dashboard [See: Functions](https://app.supabase.com/project/_/functions) we'll see our `recommendations` function in the list.  Click on the function name to get more details about the function including: Metrics, Details, Invocations, and Logs.  If we click on `Details` we can get instructions on how to `deploy` (we've already done this), and `delete` the function, how to `set` and `unset` our `secret` (which we've also already done), and how to `invoke` our function from the command line using `curl`.  This will be great for testing our new function.

The example we're given looks like this:

```sh
curl -L -X POST 'https://<project_ref_number>.functions.supabase.co/recommendations' -H 'Authorization: Bearer <project_anonymous_api_key>' --data '{"name":"Functions"}'
```
If you click the clipboard icon next to this entry, your project reference number should be conveniently filled in for you already (if not, you'll need to look up these values from your [Settings Page](https://app.supabase.com/project/_/settings/general) and fill those in.)

We'll need to change just one thing here.  The `--data` parameter is sending `name` (from the "Hello, World" function) and not the array of `movies` that we want to send.  Change it to look like this:
`--data '{"movies": ["Raiders of the Lost Ark", "Back to the Future", "Die Hard"]}'`
Use whatever movies you've enjoyed over the years.

Once you've completed your `curl` command, see the output you get:

```
{"data":" \n\nYou might enjoy other action/adventure films such as \"Indiana Jones and the Last Crusade\", \"Terminator 2: Judgment Day\", \"The Matrix\", \"Lethal Weapon\", \"The Dark Knight\", \"Mission Impossible\" series, or \"Mad Max: Fury Road\". If you're looking for something more light-hearted, try out movies like \"Groundhog Day\", \"The Princess Bride\", or \"Ferris Bueller's Day Off\".","error":null}
```

Not too bad.  OpenAI has some pretty good taste in movies!

### Conclusion

Wow, the possibilities are endless now that we have access to AI through edge functions! The hardest part will be deciding what amazing things we want to achieve with this technology. Supabase makes it easy for us by providing tools like PostgreSQL data, GoTrue authentication, cloud storage, and real-time updates, not to mention the added bonus of OpenAI edge functions. Let's brainstorm and come up with some incredible ideas for how we can use these tools to change the world!
