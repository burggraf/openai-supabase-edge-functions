#Summoning the Magical Powers of ChatGPT from your Supabase Edge Functions
ChatGPT-Powered Supabase Edge Functions

INTRO

### Seen any good movies lately?

Our sample app will collect a short list of movies that a user has enjoyed, and based on that set, we’ll recommend other movies they may also find interesting.  In short, we feed a list of movies to `OpenAI` and get back a list of recommendations.

We’ll use the `OpenAI Completions API` with the `text-davinci-003` model to get our movie recommendations from our Supabase Edge Function.  This core model, used extensively by ChatGPT, is perfect for this use case.

### Getting Started

If you don’t already have a Supabase project, you’ll want to set one up at [supabase.com](https://supabase.com).  A project on the free tier will work just fine here.

#### Set up the Supabase CLI (command line interface)

Once your project is set up, you can follow the [Edge Functions Quickstart]( https://supabase.com/docs/guides/functions/quickstart) to get your function up and running.  Specifically, you’ll need to:
-	Install the Supabase CLI (command line interface). [see guide]( https://supabase.com/docs/guides/cli)
-	Login to the CLI using the command: `supabase login`. [see guide](https://supabase.com/docs/reference/cli/usage#supabase-login)
-	Initialize Supabase inside your project using the command: `supabase init`. [see guide](https://supabase.com/docs/guides/cli/local-development#getting-started)
-	Link to your Remote Project using the command `supabase link --project-ref your-project-ref`. [see guide](https://supabase.com/docs/reference/cli/usage#supabase-link)
-	Optional: Setup your environment: Follow [this setup guide](https://deno.land/manual/getting_started/setup_your_environment) to integrate the Deno language server with your editor.

### 

#### Create the Function: `recommendations`

Following along in the Quickstart guide, you can create a function called `recommendations` to pick relevant movies.  Call it `recommenations`.  From inside your project folder, run:
`supabase functions new recommendations`

This will create a folder structure in your app that looks like this:
`supabase/functions/discovery/`
and your function will be inside the file:
`supabase/functions/discovery/index.ts`
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

This is a great starting point, but we’ll want to modify it to call the OpenAI API.
