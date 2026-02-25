# youtube_downloader
Downloads YouTube videos from YouTube website links.

Main Features
=
Download a single video or full playlist.

Output format switch: MP4 or MP3.

Quality controls:
-
MP4: max video height (best, 2160, 1440, etc.).

MP3: target bitrate (320, 256, etc.).

Optional extras:
-
Embed thumbnail into media.

Save cover image as PNG file.

Write description text file.

Write subtitles (English when available).

Metadata modes:
-
extract: keep yt-dlp extracted tags.

blank: clear tags.

custom: set your own tags with placeholders.

Live status + progress bar + log output.

Cancel support during downloads.

Settings persistence between runs.

All Buttons and Controls
=
Browse
-
Chooses output folder using folder picker.

MP4 (video) / MP3 (audio) radio buttons
-
Switch media mode and enable/disable relevant quality controls.

Max Quality combobox
-
Used in MP4 mode to cap resolution.

Audio Bitrate combobox
-
Used in MP3 mode for extract quality.

Download option checkboxes
-
Download full playlist, Embed thumbnail, Save cover PNG, Write info JSON, Write description file, Write subtitles.

Filename field
-
Currently shown disabled and forced to a safe default template (%(title).200B.%(ext)s).

Metadata Mode dropdown
-
Controls whether tags are extracted, cleared, or custom-filled.

Custom metadata fields (Title, Artist, Album, Channel / Album Artist, Comment)
-
Enabled only when mode is custom. Supports placeholders like {title}, {uploader}, {playlist_title}, {index}.

Start Download
-
Validates inputs/dependencies, saves settings, starts background worker thread.

Cancel
-
Sets a cancel flag. Worker stops at safe points via yt-dlp cancellation exception.

Progress bar + status line
-
Shows percent, bytes/speed/ETA, and playlist remaining count.

Log panel
-
Chronological events/errors/post-processing results.

Footer credit label
-
Hover interaction for author credit.

How To Navigate and Use It
=
Paste YouTube URL.

Pick save folder (Browse).

Choose MP4 or MP3.

Set quality/bitrate.

Toggle optional checkboxes as needed.

Choose metadata mode:
-
- extract for normal behavior.

- blank to strip tags.

- custom and fill tag fields/placeholders.

Click Start Download.

Watch status, progress, and logs.

Use Cancel if needed.

Close app normally to persist settings.

Validation and Dependency Rules

Requires yt-dlp.

Requires ffmpeg when MP3, thumbnail embed, or cover export is requested.

mutagen is required for blank/custom metadata editing and high-res MP3 cover replacement.

If mutagen is missing for metadata modes, app prompts whether to continue without metadata edits.

Behind-the-scenes behavior
=
Download runs in background thread to keep UI responsive.

yt-dlp options are built from UI state, including postprocessors for conversion/thumbnail/tag steps.

For playlists, app probes expected item count first and reports progress.

Metadata override and high-res cover enhancement are applied after download.
