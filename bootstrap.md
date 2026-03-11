Act as a Senior Frontend Engineer, implement a single-file HTML5 application.

Respond only with the valid, complete HTML, including CSS and JS.

* **Tech Stack:** Single-file HTML5, Modern JavaScript (ES6+), Minimal CSS.
* **Theme:** Solarized Light.
* **Icons:** Google Material Icons.
* **Code Quality:** Implement robust error handling with modal overlays, no place holders or stubs.

# 1. UI Architecture

The application consists of two main vertical regions:

## 1.1 Fixed Toolbar (Top)

A slim title bar with a persistent title and icon-only buttons:

* Floating to the left: A `settings` icon
* Floating to the right: `download`, `add_to_drive`, and `play_arrow` (Resume) icons.
* **Initial State:** Only `settings` is visible. Others are hidden or disabled until specific conditions are met.

## 1.2 Content Area (Center/Fill)

A fixed, non-scrollable container that hosts:

* **Landing Page:** Initial instructional text regarding the LLM-driven nature of the app.
* **Modals/Cards:** Centered elements with a small drop shadow for forms and messages.
* **Execution Frame:** An `<iframe>` that expands to fill the entire content area.

# 2. Functional Modules

## 2.1 Google integration

1. Use Silent Oauth2 flow with Google Identity Services for authentication.
2. Save the token in local storage and reuse it on subsequent sessions.
3. Client ID: 213106841057-jffschqelp14pa3vafc7p93e5aq454ip.apps.googleusercontent.com
4. Scopes: https://www.googleapis.com/auth/drive.appdata and https://www.googleapis.com/auth/drive.file
5. For **Google Drive** access throughout, use the token from the silent oauth2 process.
5. Use **Google Drive Picker API** for selecting files for loading and saving files to Drive.

## 2.2 Markdown sectioning

Here is the procedure for extracting, adding and replacing sections in the current markdown file.

**Extracting a section**:

1. Trim the section name and if it doesn't start with a `#`, prefix `# `.
2. Find the line in the markdown which is an exact match for the name.
3. The start of the section is from the following line.
4. The start of the next section is when a line start with a hash is found which
   has the same number or fewer leading hashes than the current section name.
5. If such a line is found, the section is considered to have all the lines until the
   starat of the next section.  If no such line is found, the section goes to the end.

**Replacing a section**:

Replacing a section follows the same appraoch as above to identify the section and
replacing all the lines in the section in the markdown file with the lines in the 
replacement text.

**Adding a section**

To add a section, trim the section name and prefix it with `# ` if it doesnt
start with a hash.  Apppend a new line + the trimmed section contents.
Insert this before the line `# Application area` in the markdown file
contents or at the end of the file if that line isn't present.

## 2.3 Settings & Configuration

Clicking the **Settings** icon disables the button and opens a modal:

* **Fields:** 
* `CODING_MODEL` (Default: `gemini-2.5-flash-lite`)
* `CODING_API_KEY` (Password/Text input)
* `CONV_MODEL` (Default: `gemini-2.5-flash-lite`)
* `CONV_API_KEY` (Password/Text input)

* **Actions:**
* **OK:** Persists values to memory and enables the **Resume** icon.
* **Load from Drive:** Fetch `config.json` from `appDataFolder` in Google Drive.
  If keys CODIMG_MODEL and CODING_API_KEY exist, populates memory and enables **Resume**.
* **Save to Drive:** Authenticates and saves current memory values to `config.json` on Google Drive.
* **Cancel:** Closes modal and re-enables the Settings button.

## 2.4 File Selection & Ingestion (Resume)

Clicking the **Resume** icon opens a dialog with the following text:

```
Please select a Markdown file either from local input or from google drive.
```

Buttons:

* **Upload:** Clicking button opens local file picker.
* **Load from Drive:** Clicking button uses **Google Drive Picker API** to select a file.

Logic:

* Upon successful file retrieval (Name + Content), trigger the **Generation Pipeline**.

## 2.5 Download button

Clicking the **Download** icon triggers a download via `URL.createObjectURL` using the loaded markdown file contents and name.

## 2.6 Add-to-Drive button

Clicking the **Add to drive** button triggers a file picker for Google Drive followed by upload.  Use the loaded markdown file and its name.

## 2.7 Window message handling

Window messages are used to communicate with the html application running in the iframe.

**Incoming messages **

1. Listen for `message` event on the window.  Ignore if `event.source` is not the iframe's window.
2. Expect event data to be `["function", args...]`
3. Dispatch to the corresponding async function in the table below.
4. Post response on iframe's window as `["functionResponse", response, error]`

**Functions Table **
- `callModel(system_prompt, user_prompt)` Calls the LLM with converstation model and key.  Result is `{ContentType, Contents, CreatedDate}` if success or error if it throws
- `replaceSection(section_name, contents)` Replaces the named section with new contents in loaded markdown.
- `addSection(section_name, contents)` Adds the named section.

# 3. The Generation Pipeline (File Loading)

Once a Markdown file is loaded, the app performs the following steps:

1. Use the section name `# Tai Chi Application`
2. Compile the markdown file to HTML as specified in `Compiling Markdown files` section.
2. Embed the generated HTML as specified in the `Embed HTML` section section.

## 3.1 Compiling Markdown Files

The following is the process to compile markdown files to HTML

1. Extract the contents of the specified section name
1. Normalize the content by trimming whitespaces at start/end.
2. Compute a SHA 256 hash of the file and create a a key: `Markdown-{SHA256}`
3. Look up local storage to see if this key exists.  If it exists, parse the JSON
     value looking for `{ContentType, Contents, CreatedDate}` and return the Contents.
4. Othewrise follow the steps in `Calling the LLM`  with `CODING_MODEL`, corresponding key and the following user prompt and empty system prompt:

```
Act as a frontend engineer.  Generate a modern single-page single-file HTML application.

REQUIREMENTS:
1. Output ONLY a complete, valid index.html file include CSS and JS.
2. Do not use place holder or stubs or TODO comments.
3. Use the send_to_user function for the output with appropriate content type.

FILE CONTENT:
{section_contents}
```

5. Save the response `{ContentTypes, Contents, CreatedDate}` into local storge and return
    the contents.


## 3.2 Calling the LLM

The following is the process to call LLMs.

If a system prompt is given, concatenate the markdown to this and use that the system prompt.
If no system prompt is given, do not use a system prompt.

If a model name and API key are provided, use that.  If not, use CONV_MODEL
and CONV_API_KEY.  

Use the specified user prompt in a request to the Open AI Endpoint.
The OpenAI call is done with **Function Calling** with the following functions:

* **send_to_user(content_type, content)** -- send the payload of the specific type to the user.  For example, to send HTML, call `send_to_user("text/html", "<html><body></body></html>")`
* **add_section(section_name, contents)** -- add a top level section to the markdown file.
* **replace_section(section_name, contents)** -- replaces the full contents of the named section with the provided contents in the loaded markdown file in memory.

Call details:
* **Endpoint:** `https://generativelanguage.googleapis.com/v1beta/openai/chat/completions`
* **Headers:** `Authorization: Bearer [API_KEY]`
* **Payload Config:**
* `model`: `[MODEL]`
* `reasoning_effort`: `medium`
* `stream`: `false`

If this succeeds and the `send_to_user` function call was made by the LLM, then
this just returns `{ContentType, Content, CreatedDate=now}`

If this succeeds without the `send_to_user` function call being made, then just
return the response text `{ContentType = "text/plain", Content = response text, CreatedDate=now}`

Otherwise return an appropriate error.

 
## 3.4 Embed HTML 

1. Inject the resulting HTML string into the `srcdoc` attribute of the main content iframe.
2. **UI Update:** Enable the `add_to_drive` and `download` buttons; disable the `resume` button.
3. **On Failure:** Display the error message in a modal within the content area.

