# Magic Mirror Compliments for TRMNL

A TRMNL recipe that brings the classic "Magic Mirror" compliments logic directly to your e-ink display. The plugin intelligently cycles through dynamic quotes and compliments based on the current time of day (Morning, Afternoon, Evening), randomly sampling from the available pool just like the original Magic Mirror module.

![Magic Mirror Preview](Magic Mirror Example.png)

## For Users: How to Use

If adding this from the TRMNL Community Recipes, simply add the plugin to your dashboard and configure the required fields:

1. **Magic Mirror JSON URL**: You must provide a **raw** link to a valid JSON file containing your compliments. 
   - *Example:* `https://raw.githubusercontent.com/MichMich/MagicMirror/master/modules/default/compliments/compliments.json`
   - *Note:* Do not use standard GitHub web links (containing `/blob/`), as the HTML wrapper will crash the device parsing.
2. **Local Time Zone**: Select your local timezone. This is critical for the device to know whether it is currently morning, afternoon, or evening.

## For Developers: Manual / Private Installation

If you want to test this recipe yourself, fork it, or manually deploy it as a Private Plugin on your TRMNL dashboard, follow these exact steps:

### 1. Create the Plugin
1. Log into your TRMNL dashboard and navigate to the Plugins tab.
2. Click **Add Private Plugin** (or "New Private Plugin").
3. Fill out the configuration fields exactly as follows:
   - **Name:** Magic Mirror Compliments (or whatever you prefer)
   - **Strategy:** Polling
   - **Polling URL(s):** `{{ json_url }}` *(TRMNL will dynamically replace this variable with the user's config)*
   - **Polling Verb:** GET
   - **Form Fields:** Copy and paste the entire contents of the `variables.yml` file from this repository.
   - **Remove bleed margin?:** No
   - **Enable Dark Mode?:** No
   - **Framework CSS version:** v3.0.2 (or whatever the active default is)

### 2. Add the Code
1. Scroll down to the **Markup (Liquid) Template** editor.
2. Copy the entire contents of `markup.html` from this repository and paste it into the editor.
3. Click **Save**.

### 3. Test on Your Device
1. Once saved, add the newly created Private Plugin to your device's playlist.
2. Configure it with a raw JSON URL and your local timezone.
3. Click **Force Refresh** on the plugin settings page to immediately generate the screen on your device or in the dashboard simulator. 

## Technical Details

- **Responsive Typography:** The plugin parses the length of your sampled quote. Shorter quotes will render larger (up to `text-6xl`), while long quotes will gracefully scale down to fit the screen bounds.
- **Fail-safes:** It automatically protects against missing time bounds, falling back to an `"anytime"` array block if provided in your JSON payload. It also prevents infinite loop rendering limits inherent in constrained liquid parsing servers.
- **Strict Logic:** Safely ignores complex nested object nodes in community JSON files, only extracting standard Top-Level Array Strings.
