#!/usr/bin/env python3

from __future__ import annotations

import json
import os
import shutil
import threading
import tkinter as tk
import urllib.error
import urllib.request
from dataclasses import dataclass
from pathlib import Path
from tkinter import filedialog, messagebox, ttk
from typing import Any

DEFAULT_FILENAME_TEMPLATE = "%(title).200B.%(ext)s"

try:
    import yt_dlp
except Exception:  # pragma: no cover - runtime dependency check
    yt_dlp = None

try:
    from mutagen.id3 import APIC, COMM, TALB, TIT2, TPE1, TPE2, ID3
    from mutagen.mp3 import MP3
    from mutagen.mp4 import MP4

    MUTAGEN_AVAILABLE = True
except Exception:  # pragma: no cover - optional dependency
    MUTAGEN_AVAILABLE = False


@dataclass
class DownloadSettings:
    url: str
    output_dir: str
    media_format: str
    quality: str
    audio_bitrate: str
    download_playlist: bool
    embed_thumbnail: bool
    save_cover_png: bool
    metadata_mode: str
    filename_template: str
    write_info_json: bool
    write_description: bool
    write_subtitles: bool
    custom_title: str
    custom_artist: str
    custom_album: str
    custom_channel: str
    custom_comment: str


class SafeDict(dict):
    def __missing__(self, key: str) -> str:
        return ""


def format_bytes(num: float | None) -> str:
    if not num:
        return "0 B"
    value = float(num)
    units = ["B", "KB", "MB", "GB", "TB"]
    for unit in units:
        if value < 1024.0:
            return f"{value:.1f} {unit}"
        value /= 1024.0
    return f"{value:.1f} PB"


def flatten_entries(info: dict[str, Any]) -> list[dict[str, Any]]:
    if not isinstance(info, dict):
        return []
    entries = info.get("entries")
    if not entries:
        return [info]
    flat: list[dict[str, Any]] = []
    for entry in entries:
        if not entry:
            continue
        flat.extend(flatten_entries(entry))
    return flat


class YouTubeDownloaderApp(tk.Tk):
    def __init__(self) -> None:
        super().__init__()
        self.title("YouTube to MP4/MP3 Downloader")
        self.geometry("940x840")
        self.minsize(900, 680)

        self.download_thread: threading.Thread | None = None
        self.cancel_event = threading.Event()
        self.total_download_items = 0
        self.completed_download_items = 0
        self._credit_hover_job: str | None = None

        self._build_variables()
        self._load_saved_settings()
        self._build_ui()
        self._refresh_format_fields()
        self._refresh_custom_metadata_fields()
        self.protocol("WM_DELETE_WINDOW", self._on_close)

    def _build_variables(self) -> None:
        self.url_var = tk.StringVar()
        self.output_dir_var = tk.StringVar(value=str(Path.home() / "Downloads"))
        self.media_format_var = tk.StringVar(value="mp4")
        self.quality_var = tk.StringVar(value="best")
        self.audio_bitrate_var = tk.StringVar(value="320")
        self.download_playlist_var = tk.BooleanVar(value=False)
        self.embed_thumbnail_var = tk.BooleanVar(value=True)
        self.save_cover_png_var = tk.BooleanVar(value=False)
        self.metadata_mode_var = tk.StringVar(value="extract")
        self.filename_template_var = tk.StringVar(value=DEFAULT_FILENAME_TEMPLATE)
        self.write_info_json_var = tk.BooleanVar(value=False)
        self.write_description_var = tk.BooleanVar(value=False)
        self.write_subtitles_var = tk.BooleanVar(value=False)
        self.status_var = tk.StringVar(value="Ready")
        self.progress_var = tk.DoubleVar(value=0.0)

        self.custom_title_var = tk.StringVar()
        self.custom_artist_var = tk.StringVar()
        self.custom_album_var = tk.StringVar()
        self.custom_channel_var = tk.StringVar()
        self.custom_comment_var = tk.StringVar()

    def _settings_path(self) -> Path:
        return Path(__file__).with_name("youtube_downloader_settings.json")

    def _collect_settings_payload(self) -> dict[str, Any]:
        return {
            "url": self.url_var.get(),
            "output_dir": self.output_dir_var.get(),
            "media_format": self.media_format_var.get(),
            "quality": self.quality_var.get(),
            "audio_bitrate": self.audio_bitrate_var.get(),
            "download_playlist": self.download_playlist_var.get(),
            "embed_thumbnail": self.embed_thumbnail_var.get(),
            "save_cover_png": self.save_cover_png_var.get(),
            "metadata_mode": self.metadata_mode_var.get(),
            "filename_template": self.filename_template_var.get(),
            "write_info_json": self.write_info_json_var.get(),
            "write_description": self.write_description_var.get(),
            "write_subtitles": self.write_subtitles_var.get(),
            "custom_title": self.custom_title_var.get(),
            "custom_artist": self.custom_artist_var.get(),
            "custom_album": self.custom_album_var.get(),
            "custom_channel": self.custom_channel_var.get(),
            "custom_comment": self.custom_comment_var.get(),
        }

    def _save_settings(self) -> None:
        try:
            payload = self._collect_settings_payload()
            self._settings_path().write_text(json.dumps(payload, indent=2), encoding="utf-8")
        except Exception as exc:
            self.log(f"Could not save settings: {exc}")

    def _load_saved_settings(self) -> None:
        path = self._settings_path()
        if not path.exists():
            return
        try:
            data = json.loads(path.read_text(encoding="utf-8"))
            if not isinstance(data, dict):
                return
        except Exception:
            return

        str_fields = {
            "url": self.url_var,
            "output_dir": self.output_dir_var,
            "quality": self.quality_var,
            "audio_bitrate": self.audio_bitrate_var,
            "filename_template": self.filename_template_var,
            "custom_title": self.custom_title_var,
            "custom_artist": self.custom_artist_var,
            "custom_album": self.custom_album_var,
            "custom_channel": self.custom_channel_var,
            "custom_comment": self.custom_comment_var,
        }
        bool_fields = {
            "download_playlist": self.download_playlist_var,
            "embed_thumbnail": self.embed_thumbnail_var,
            "save_cover_png": self.save_cover_png_var,
            "write_info_json": self.write_info_json_var,
            "write_description": self.write_description_var,
            "write_subtitles": self.write_subtitles_var,
        }

        for key, var in str_fields.items():
            value = data.get(key)
            if isinstance(value, str):
                var.set(value)
        for key, var in bool_fields.items():
            value = data.get(key)
            if isinstance(value, bool):
                var.set(value)

        media_format = data.get("media_format")
        if media_format in {"mp3", "mp4"}:
            self.media_format_var.set(media_format)

        metadata_mode = data.get("metadata_mode")
        if metadata_mode in {"extract", "blank", "custom"}:
            self.metadata_mode_var.set(metadata_mode)

        # Force clean title-only output names.
        self.filename_template_var.set(DEFAULT_FILENAME_TEMPLATE)

    def _on_close(self) -> None:
        self._clear_credit_hover_job()
        self._save_settings()
        if self.download_thread and self.download_thread.is_alive():
            close_now = messagebox.askyesno(
                "Download running",
                "A download is still running. Close and cancel it?",
            )
            if not close_now:
                return
            self.cancel_event.set()
        self.destroy()

    def _build_ui(self) -> None:
        container = ttk.Frame(self, padding=12)
        container.pack(fill=tk.BOTH, expand=True)
        container.columnconfigure(1, weight=1)

        row = 0
        ttk.Label(container, text="YouTube URL").grid(row=row, column=0, sticky="w", padx=(0, 8), pady=4)
        self.url_entry = ttk.Entry(container, textvariable=self.url_var)
        self.url_entry.grid(row=row, column=1, sticky="ew", pady=4)
        row += 1

        ttk.Label(container, text="Save Directory").grid(row=row, column=0, sticky="w", padx=(0, 8), pady=4)
        self.output_entry = ttk.Entry(container, textvariable=self.output_dir_var)
        self.output_entry.grid(row=row, column=1, sticky="ew", pady=4)
        ttk.Button(container, text="Browse", command=self._browse_output).grid(row=row, column=2, sticky="ew", padx=(8, 0), pady=4)
        row += 1

        format_frame = ttk.LabelFrame(container, text="Format", padding=10)
        format_frame.grid(row=row, column=0, columnspan=3, sticky="ew", pady=8)
        format_frame.columnconfigure(4, weight=1)

        ttk.Radiobutton(
            format_frame,
            text="MP4 (video)",
            variable=self.media_format_var,
            value="mp4",
            command=self._refresh_format_fields,
        ).grid(row=0, column=0, sticky="w", padx=(0, 14))

        ttk.Radiobutton(
            format_frame,
            text="MP3 (audio)",
            variable=self.media_format_var,
            value="mp3",
            command=self._refresh_format_fields,
        ).grid(row=0, column=1, sticky="w", padx=(0, 20))

        self.quality_label = ttk.Label(format_frame, text="Max Quality")
        self.quality_label.grid(row=0, column=2, sticky="e")
        self.quality_combo = ttk.Combobox(
            format_frame,
            textvariable=self.quality_var,
            values=["best", "2160", "1440", "1080", "720", "480", "360"],
            state="readonly",
            width=10,
        )
        self.quality_combo.grid(row=0, column=3, sticky="w", padx=(8, 20))

        self.bitrate_label = ttk.Label(format_frame, text="Audio Bitrate")
        self.bitrate_label.grid(row=0, column=4, sticky="e")
        self.bitrate_combo = ttk.Combobox(
            format_frame,
            textvariable=self.audio_bitrate_var,
            values=["320", "256", "192", "160", "128"],
            state="readonly",
            width=10,
        )
        self.bitrate_combo.grid(row=0, column=5, sticky="w", padx=(8, 0))

        row += 1
        options_frame = ttk.LabelFrame(container, text="Download Options", padding=10)
        options_frame.grid(row=row, column=0, columnspan=3, sticky="ew", pady=8)
        options_frame.columnconfigure(0, weight=1)
        options_frame.columnconfigure(1, weight=1)

        ttk.Checkbutton(
            options_frame,
            text="Download full playlist",
            variable=self.download_playlist_var,
        ).grid(row=0, column=0, sticky="w", pady=2)

        ttk.Checkbutton(
            options_frame,
            text="Embed thumbnail",
            variable=self.embed_thumbnail_var,
        ).grid(row=0, column=1, sticky="w", pady=2)

        ttk.Checkbutton(
            options_frame,
            text="Save cover PNG (extra file)",
            variable=self.save_cover_png_var,
        ).grid(row=1, column=0, sticky="w", pady=2)

        ttk.Checkbutton(
            options_frame,
            text="Write info JSON",
            variable=self.write_info_json_var,
        ).grid(row=1, column=1, sticky="w", pady=2)

        ttk.Checkbutton(
            options_frame,
            text="Write description file",
            variable=self.write_description_var,
        ).grid(row=2, column=0, sticky="w", pady=2)

        ttk.Checkbutton(
            options_frame,
            text="Write subtitles (English when available)",
            variable=self.write_subtitles_var,
        ).grid(row=2, column=1, sticky="w", pady=2)

        row += 1
        ttk.Label(container, text="Filename").grid(row=row, column=0, sticky="w", padx=(0, 8), pady=4)
        self.template_entry = ttk.Entry(container, textvariable=self.filename_template_var, state="disabled")
        self.template_entry.grid(row=row, column=1, columnspan=2, sticky="ew", pady=4)
        row += 1

        metadata_frame = ttk.LabelFrame(container, text="Metadata", padding=10)
        metadata_frame.grid(row=row, column=0, columnspan=3, sticky="ew", pady=8)
        metadata_frame.columnconfigure(1, weight=1)

        ttk.Label(metadata_frame, text="Mode").grid(row=0, column=0, sticky="w")
        self.metadata_combo = ttk.Combobox(
            metadata_frame,
            textvariable=self.metadata_mode_var,
            state="readonly",
            values=["extract", "blank", "custom"],
            width=12,
        )
        self.metadata_combo.grid(row=0, column=1, sticky="w", padx=(8, 0), pady=(0, 6))
        self.metadata_combo.bind("<<ComboboxSelected>>", lambda _: self._refresh_custom_metadata_fields())

        self.custom_frame = ttk.Frame(metadata_frame)
        self.custom_frame.grid(row=1, column=0, columnspan=2, sticky="ew")
        self.custom_frame.columnconfigure(1, weight=1)

        self.custom_entries: list[ttk.Entry] = []
        custom_fields = [
            ("Title", self.custom_title_var),
            ("Artist", self.custom_artist_var),
            ("Album", self.custom_album_var),
            ("Channel / Album Artist", self.custom_channel_var),
            ("Comment", self.custom_comment_var),
        ]
        for idx, (label, var) in enumerate(custom_fields):
            ttk.Label(self.custom_frame, text=label).grid(row=idx, column=0, sticky="w", pady=2, padx=(0, 8))
            entry = ttk.Entry(self.custom_frame, textvariable=var)
            entry.grid(row=idx, column=1, sticky="ew", pady=2)
            self.custom_entries.append(entry)

        ttk.Label(
            self.custom_frame,
            text="In custom mode, empty fields auto-extract per video. Filled fields apply to every downloaded item.",
            foreground="#555555",
        ).grid(row=len(custom_fields), column=0, columnspan=2, sticky="w", pady=(6, 0))
        ttk.Label(
            self.custom_frame,
            text="Supports placeholders: {title}, {uploader}, {channel}, {playlist_title}, {index}.",
            foreground="#555555",
        ).grid(row=len(custom_fields) + 1, column=0, columnspan=2, sticky="w", pady=(2, 0))

        row += 1
        action_frame = ttk.Frame(container)
        action_frame.grid(row=row, column=0, columnspan=3, sticky="ew", pady=8)
        action_frame.columnconfigure(0, weight=1)

        self.start_btn = ttk.Button(action_frame, text="Start Download", command=self.start_download)
        self.start_btn.grid(row=0, column=0, sticky="w")

        self.cancel_btn = ttk.Button(action_frame, text="Cancel", command=self.cancel_download, state=tk.DISABLED)
        self.cancel_btn.grid(row=0, column=1, sticky="w", padx=(8, 0))

        row += 1
        self.progress = ttk.Progressbar(container, variable=self.progress_var, maximum=100)
        self.progress.grid(row=row, column=0, columnspan=3, sticky="ew", pady=(4, 2))

        row += 1
        ttk.Label(container, textvariable=self.status_var).grid(row=row, column=0, columnspan=3, sticky="w", pady=(0, 6))

        row += 1
        log_frame = ttk.LabelFrame(container, text="Log", padding=6)
        log_frame.grid(row=row, column=0, columnspan=3, sticky="nsew")
        container.rowconfigure(row, weight=1)
        log_frame.rowconfigure(0, weight=1)
        log_frame.columnconfigure(0, weight=1)

        self.log_text = tk.Text(log_frame, wrap=tk.WORD, height=14)
        self.log_text.grid(row=0, column=0, sticky="nsew")
        scroll = ttk.Scrollbar(log_frame, command=self.log_text.yview)
        scroll.grid(row=0, column=1, sticky="ns")
        self.log_text.configure(yscrollcommand=scroll.set)

        row += 1
        self.credit_var = tk.StringVar(value="ZeroG")
        self.credit_label = ttk.Label(container, textvariable=self.credit_var, anchor="center")
        self.credit_label.grid(row=row, column=0, columnspan=3, sticky="ew", pady=(8, 0))
        self.credit_label.bind("<Enter>", self._on_credit_hover_start)
        self.credit_label.bind("<Leave>", self._on_credit_hover_end)

    def _browse_output(self) -> None:
        selected = filedialog.askdirectory(initialdir=self.output_dir_var.get() or str(Path.home()))
        if selected:
            self.output_dir_var.set(selected)

    def _refresh_format_fields(self) -> None:
        is_mp3 = self.media_format_var.get() == "mp3"
        if is_mp3:
            self.quality_label.configure(state=tk.DISABLED)
            self.quality_combo.configure(state=tk.DISABLED)
            self.bitrate_label.configure(state=tk.NORMAL)
            self.bitrate_combo.configure(state="readonly")
        else:
            self.quality_label.configure(state=tk.NORMAL)
            self.quality_combo.configure(state="readonly")
            self.bitrate_label.configure(state=tk.DISABLED)
            self.bitrate_combo.configure(state=tk.DISABLED)

    def _refresh_custom_metadata_fields(self) -> None:
        mode = self.metadata_mode_var.get()
        state = tk.NORMAL if mode == "custom" else tk.DISABLED
        for entry in self.custom_entries:
            entry.configure(state=state)

    def _set_status(self, text: str) -> None:
        self.status_var.set(text)

    def _append_log(self, line: str) -> None:
        self.log_text.insert(tk.END, line.rstrip() + "\n")
        self.log_text.see(tk.END)

    def log(self, line: str) -> None:
        self.after(0, self._append_log, line)

    def _set_status_threadsafe(self, text: str) -> None:
        self.after(0, self._set_status, text)

    def _set_progress_threadsafe(self, value: float) -> None:
        self.after(0, self.progress_var.set, max(0.0, min(100.0, value)))

    def _on_credit_hover_start(self, _event: object = None) -> None:
        self._clear_credit_hover_job()
        self._credit_hover_job = self.after(800, self._show_credit_full_name)

    def _on_credit_hover_end(self, _event: object = None) -> None:
        self._clear_credit_hover_job()
        self.credit_var.set("ZeroG")

    def _show_credit_full_name(self) -> None:
        self._credit_hover_job = None
        self.credit_var.set("Ziad saadawi (ZeroG_G)")

    def _clear_credit_hover_job(self) -> None:
        if self._credit_hover_job is None:
            return
        try:
            self.after_cancel(self._credit_hover_job)
        except tk.TclError:
            pass
        self._credit_hover_job = None

    def _set_running_state(self, running: bool) -> None:
        start_state = tk.DISABLED if running else tk.NORMAL
        cancel_state = tk.NORMAL if running else tk.DISABLED
        self.start_btn.configure(state=start_state)
        self.cancel_btn.configure(state=cancel_state)

    def _collect_settings(self) -> DownloadSettings:
        return DownloadSettings(
            url=self.url_var.get().strip(),
            output_dir=self.output_dir_var.get().strip(),
            media_format=self.media_format_var.get(),
            quality=self.quality_var.get(),
            audio_bitrate=self.audio_bitrate_var.get(),
            download_playlist=self.download_playlist_var.get(),
            embed_thumbnail=self.embed_thumbnail_var.get(),
            save_cover_png=self.save_cover_png_var.get(),
            metadata_mode=self.metadata_mode_var.get(),
            filename_template=self.filename_template_var.get().strip() or DEFAULT_FILENAME_TEMPLATE,
            write_info_json=self.write_info_json_var.get(),
            write_description=self.write_description_var.get(),
            write_subtitles=self.write_subtitles_var.get(),
            custom_title=self.custom_title_var.get(),
            custom_artist=self.custom_artist_var.get(),
            custom_album=self.custom_album_var.get(),
            custom_channel=self.custom_channel_var.get(),
            custom_comment=self.custom_comment_var.get(),
        )

    def _validate_settings(self, settings: DownloadSettings) -> bool:
        if yt_dlp is None:
            messagebox.showerror(
                "Missing dependency",
                "yt-dlp is not installed.\nInstall with:\n\npip install yt-dlp",
            )
            return False

        if not settings.url:
            messagebox.showerror("Missing URL", "Please paste a YouTube URL.")
            return False

        if not settings.output_dir:
            messagebox.showerror("Missing folder", "Please choose a save directory.")
            return False

        try:
            Path(settings.output_dir).mkdir(parents=True, exist_ok=True)
        except Exception as exc:
            messagebox.showerror("Invalid folder", f"Cannot access output directory:\n{exc}")
            return False

        ffmpeg_required = settings.media_format == "mp3" or settings.embed_thumbnail or settings.save_cover_png
        if ffmpeg_required and shutil.which("ffmpeg") is None:
            messagebox.showerror(
                "Missing ffmpeg",
                "ffmpeg is required for selected options.\nPlease install ffmpeg and ensure it is in PATH.",
            )
            return False

        if settings.metadata_mode in {"blank", "custom"} and not MUTAGEN_AVAILABLE:
            proceed = messagebox.askyesno(
                "mutagen not installed",
                "Custom/blank metadata editing needs mutagen.\n"
                "Continue without metadata edits?\n\nInstall with: pip install mutagen",
            )
            if not proceed:
                return False
        return True

    def start_download(self) -> None:
        if self.download_thread and self.download_thread.is_alive():
            return

        settings = self._collect_settings()
        if not self._validate_settings(settings):
            return
        self._save_settings()

        self.log("Starting download...")
        self._set_progress_threadsafe(0.0)
        self._set_status("Preparing download...")
        self.cancel_event.clear()
        self._set_running_state(True)

        self.download_thread = threading.Thread(
            target=self._download_worker,
            args=(settings,),
            daemon=True,
        )
        self.download_thread.start()

    def cancel_download(self) -> None:
        if self.download_thread and self.download_thread.is_alive():
            self.cancel_event.set()
            self.log("Cancel requested. Stopping after current operation...")

    def _progress_hook(self, data: dict[str, Any]) -> None:
        if self.cancel_event.is_set():
            raise yt_dlp.utils.DownloadCancelled("Cancelled by user")

        status = data.get("status")
        if status == "downloading":
            downloaded = data.get("downloaded_bytes", 0.0)
            total = data.get("total_bytes") or data.get("total_bytes_estimate") or 0.0
            speed = data.get("speed")
            eta = data.get("eta")

            pct = (downloaded / total * 100.0) if total else 0.0
            self._set_progress_threadsafe(pct)

            status_text = (
                f"Downloading {pct:.1f}% | {format_bytes(downloaded)} / {format_bytes(total)} | "
                f"{format_bytes(speed)}/s | ETA: {eta if eta is not None else '-'}s"
            )
            if self.total_download_items > 1:
                remaining = max(self.total_download_items - self.completed_download_items, 0)
                status_text += f" | Files left: {remaining}"
            self._set_status_threadsafe(status_text)
        elif status == "finished":
            self.completed_download_items += 1
            remaining = max(self.total_download_items - self.completed_download_items, 0)
            self._set_status_threadsafe("Download complete, processing...")
            filename = data.get("filename")
            if filename:
                self.log(f"Finished: {Path(filename).name}")
            if self.total_download_items > 1:
                self.log(
                    f"Playlist progress: {self.completed_download_items}/{self.total_download_items} complete, {remaining} left."
                )

    def _build_ydl_options(self, settings: DownloadSettings) -> dict[str, Any]:
        outtmpl = os.path.join(settings.output_dir, DEFAULT_FILENAME_TEMPLATE)
        ydl_opts: dict[str, Any] = {
            "outtmpl": outtmpl,
            "noplaylist": not settings.download_playlist,
            "ignoreerrors": True,
            "quiet": True,
            "no_warnings": True,
            "progress_hooks": [self._progress_hook],
            "writethumbnail": settings.embed_thumbnail or settings.save_cover_png,
            "keepthumbnail": settings.save_cover_png,
            "writeinfojson": settings.write_info_json,
            "writedescription": settings.write_description,
            "writesubtitles": settings.write_subtitles,
            "writeautomaticsub": settings.write_subtitles,
            "subtitleslangs": ["en.*", "en"],
            "overwrites": False,
            "addmetadata": settings.metadata_mode == "extract",
        }

        postprocessors: list[dict[str, Any]] = []

        if settings.media_format == "mp3":
            ydl_opts["format"] = "bestaudio/best"
            postprocessors.append(
                {
                    "key": "FFmpegExtractAudio",
                    "preferredcodec": "mp3",
                    "preferredquality": settings.audio_bitrate,
                }
            )
            if settings.embed_thumbnail:
                postprocessors.append({"key": "FFmpegThumbnailsConvertor", "format": "jpg"})
                postprocessors.append({"key": "EmbedThumbnail"})
                ydl_opts["postprocessor_args"] = {
                    "EmbedThumbnail+ffmpeg_o": ["-id3v2_version", "3"],
                }
            if settings.save_cover_png:
                postprocessors.append({"key": "FFmpegThumbnailsConvertor", "format": "png"})
        else:
            if settings.quality == "best":
                ydl_opts["format"] = "bv*+ba/b"
            else:
                max_height = int(settings.quality)
                ydl_opts["format"] = (
                    f"bv*[height<={max_height}]+ba/b[height<={max_height}]"
                    f"/best[height<={max_height}]"
                )
            ydl_opts["merge_output_format"] = "mp4"

        if settings.embed_thumbnail and settings.media_format != "mp3":
            postprocessors.append({"key": "EmbedThumbnail"})
        if settings.save_cover_png and settings.media_format != "mp3":
            postprocessors.append({"key": "FFmpegThumbnailsConvertor", "format": "png"})

        if postprocessors:
            ydl_opts["postprocessors"] = postprocessors

        return ydl_opts

    def _count_expected_items(self, settings: DownloadSettings) -> int:
        try:
            probe_opts = {
                "quiet": True,
                "no_warnings": True,
                "ignoreerrors": True,
                "noplaylist": not settings.download_playlist,
                "extract_flat": "in_playlist",
                "skip_download": True,
            }
            with yt_dlp.YoutubeDL(probe_opts) as probe:
                info = probe.extract_info(settings.url, download=False)
            if not info:
                return 1
            entries = flatten_entries(info)
            return max(len(entries), 1)
        except Exception:
            return 1

    def _derive_output_path(
        self,
        ydl: yt_dlp.YoutubeDL,
        entry: dict[str, Any],
        media_format: str,
    ) -> Path | None:
        requested_downloads = entry.get("requested_downloads") or []
        for item in requested_downloads:
            candidate = item.get("filepath")
            if candidate and Path(candidate).exists():
                return Path(candidate)

        filename = entry.get("_filename")
        if filename and Path(filename).exists():
            path = Path(filename)
            if media_format == "mp3":
                mp3_path = path.with_suffix(".mp3")
                if mp3_path.exists():
                    return mp3_path
            return path

        try:
            prepared = Path(ydl.prepare_filename(entry))
            if media_format == "mp3":
                prepared = prepared.with_suffix(".mp3")
            elif media_format == "mp4":
                prepared = prepared.with_suffix(".mp4")
            if prepared.exists():
                return prepared
        except Exception:
            pass

        return None

    def _render_custom_field(self, template: str, context: dict[str, Any]) -> str:
        text = template.strip()
        if not text:
            return ""
        try:
            return text.format_map(SafeDict(context)).strip()
        except Exception:
            return text

    def _build_custom_metadata(self, settings: DownloadSettings, entry: dict[str, Any], index: int, top_info: dict[str, Any]) -> dict[str, str]:
        ctx = dict(entry)
        ctx.setdefault("index", index + 1)
        playlist_title = top_info.get("title", "") or ""
        channel_name = entry.get("channel") or entry.get("uploader") or ""
        ctx.setdefault("playlist_title", playlist_title)
        ctx.setdefault("channel", channel_name)

        extracted_defaults = {
            "title": str(entry.get("track") or entry.get("title") or ""),
            "artist": str(entry.get("artist") or entry.get("uploader") or ""),
            "album": str(entry.get("album") or playlist_title or ""),
            "album_artist": str(channel_name),
            "comment": str(entry.get("webpage_url") or ""),
        }

        custom_templates = {
            "title": settings.custom_title,
            "artist": settings.custom_artist,
            "album": settings.custom_album,
            "album_artist": settings.custom_channel,
            "comment": settings.custom_comment,
        }

        metadata: dict[str, str] = {}
        for key, template in custom_templates.items():
            rendered = self._render_custom_field(template, ctx)
            metadata[key] = rendered if rendered else extracted_defaults.get(key, "")
        return {k: v for k, v in metadata.items() if v}

    def _apply_mp3_metadata(self, path: Path, metadata: dict[str, str], clear_only: bool) -> None:
        audio = MP3(path, ID3=ID3)
        if audio.tags is None:
            audio.add_tags()
        tags = audio.tags

        for frame_key in ("TIT2", "TPE1", "TALB", "TPE2", "COMM"):
            tags.delall(frame_key)

        if not clear_only:
            if metadata.get("title"):
                tags.add(TIT2(encoding=3, text=metadata["title"]))
            if metadata.get("artist"):
                tags.add(TPE1(encoding=3, text=metadata["artist"]))
            if metadata.get("album"):
                tags.add(TALB(encoding=3, text=metadata["album"]))
            if metadata.get("album_artist"):
                tags.add(TPE2(encoding=3, text=metadata["album_artist"]))
            if metadata.get("comment"):
                tags.add(COMM(encoding=3, lang="eng", desc="", text=metadata["comment"]))

        audio.save()

    def _apply_mp4_metadata(self, path: Path, metadata: dict[str, str], clear_only: bool) -> None:
        audio = MP4(path)
        if audio.tags is None:
            audio.add_tags()

        tag_map = {
            "title": "\xa9nam",
            "artist": "\xa9ART",
            "album": "\xa9alb",
            "album_artist": "aART",
            "comment": "\xa9cmt",
        }

        for key in tag_map.values():
            audio.tags.pop(key, None)

        if not clear_only:
            for src_key, target_key in tag_map.items():
                value = metadata.get(src_key)
                if value:
                    audio.tags[target_key] = [value]

        audio.save()

    def _guess_image_mime(self, image_bytes: bytes) -> str:
        if image_bytes.startswith(b"\xff\xd8\xff"):
            return "image/jpeg"
        if image_bytes.startswith(b"\x89PNG\r\n\x1a\n"):
            return "image/png"
        if image_bytes.startswith(b"RIFF") and b"WEBP" in image_bytes[:16]:
            return "image/webp"
        return "image/jpeg"

    def _download_thumbnail_bytes(self, url: str) -> bytes | None:
        try:
            req = urllib.request.Request(
                url,
                headers={"User-Agent": "Mozilla/5.0"},
            )
            with urllib.request.urlopen(req, timeout=12) as resp:
                data = resp.read()
            if not data or len(data) < 1024:
                return None
            return data
        except (urllib.error.URLError, TimeoutError, ValueError):
            return None

    def _best_thumbnail_urls(self, entry: dict[str, Any]) -> list[str]:
        urls: list[str] = []
        video_id = str(entry.get("id") or "").strip()
        if video_id:
            urls.extend(
                [
                    f"https://i.ytimg.com/vi/{video_id}/maxresdefault.jpg",
                    f"https://i.ytimg.com/vi/{video_id}/sddefault.jpg",
                    f"https://i.ytimg.com/vi/{video_id}/hqdefault.jpg",
                ]
            )

        thumbs = entry.get("thumbnails") or []
        ranked: list[tuple[int, int, str]] = []
        for thumb in thumbs:
            if not isinstance(thumb, dict):
                continue
            thumb_url = str(thumb.get("url") or "").strip()
            if not thumb_url:
                continue
            width = int(thumb.get("width") or 0)
            height = int(thumb.get("height") or 0)
            area = width * height
            # Prefer JPEG/PNG when sizes are similar for better player compatibility.
            is_preferred_ext = 1 if any(ext in thumb_url.lower() for ext in (".jpg", ".jpeg", ".png")) else 0
            ranked.append((area, is_preferred_ext, thumb_url))

        for _, __, thumb_url in sorted(ranked, key=lambda t: (t[0], t[1]), reverse=True):
            urls.append(thumb_url)

        deduped: list[str] = []
        seen: set[str] = set()
        for u in urls:
            if u and u not in seen:
                seen.add(u)
                deduped.append(u)
        return deduped

    def _apply_high_res_cover_art_mp3(
        self,
        settings: DownloadSettings,
        top_info: dict[str, Any],
        ydl: yt_dlp.YoutubeDL,
    ) -> None:
        if settings.media_format != "mp3" or not settings.embed_thumbnail:
            return
        if not MUTAGEN_AVAILABLE:
            self.log("High-res cover enhancement skipped: mutagen is not installed.")
            return

        entries = flatten_entries(top_info)
        if not entries:
            return

        updated_count = 0
        for entry in entries:
            file_path = self._derive_output_path(ydl, entry, settings.media_format)
            if not file_path or file_path.suffix.lower() != ".mp3":
                continue

            image_bytes: bytes | None = None
            for thumb_url in self._best_thumbnail_urls(entry):
                image_bytes = self._download_thumbnail_bytes(thumb_url)
                if image_bytes:
                    break
            if not image_bytes:
                continue

            try:
                audio = MP3(file_path, ID3=ID3)
                if audio.tags is None:
                    audio.add_tags()
                audio.tags.delall("APIC")
                audio.tags.add(
                    APIC(
                        encoding=3,
                        mime=self._guess_image_mime(image_bytes),
                        type=3,
                        desc="Cover",
                        data=image_bytes,
                    )
                )
                audio.save(v2_version=3)
                updated_count += 1
            except Exception as exc:
                self.log(f"High-res cover embedding failed for {file_path.name}: {exc}")

        if updated_count > 0:
            self.log(f"Upgraded embedded cover art for {updated_count} MP3 file(s).")

    def _apply_metadata_overrides(
        self,
        settings: DownloadSettings,
        top_info: dict[str, Any],
        ydl: yt_dlp.YoutubeDL,
    ) -> None:
        if settings.metadata_mode not in {"blank", "custom"}:
            return
        if not MUTAGEN_AVAILABLE:
            self.log("Metadata override skipped: mutagen is not installed.")
            return

        clear_only = settings.metadata_mode == "blank"
        entries = flatten_entries(top_info)
        if not entries:
            return

        updated_count = 0
        for idx, entry in enumerate(entries):
            file_path = self._derive_output_path(ydl, entry, settings.media_format)
            if not file_path:
                continue

            try:
                metadata = {} if clear_only else self._build_custom_metadata(settings, entry, idx, top_info)
                if settings.media_format == "mp3":
                    self._apply_mp3_metadata(file_path, metadata, clear_only=clear_only)
                else:
                    self._apply_mp4_metadata(file_path, metadata, clear_only=clear_only)
                updated_count += 1
            except Exception as exc:
                self.log(f"Metadata update failed for {file_path.name}: {exc}")

        if clear_only:
            self.log(f"Cleared metadata for {updated_count} file(s).")
        else:
            self.log(f"Applied custom metadata to {updated_count} file(s).")

    def _download_worker(self, settings: DownloadSettings) -> None:
        try:
            options = self._build_ydl_options(settings)
            self.log("yt-dlp options prepared.")
            self.log(f"Format: {settings.media_format.upper()} | Playlist mode: {settings.download_playlist}")
            self.total_download_items = self._count_expected_items(settings)
            self.completed_download_items = 0
            if self.total_download_items > 1:
                self.log(f"Total playlist items to download: {self.total_download_items}")

            with yt_dlp.YoutubeDL(options) as ydl:
                info = ydl.extract_info(settings.url, download=True)
                if info is None:
                    raise RuntimeError("Download failed. No information returned by yt-dlp.")
                self._apply_metadata_overrides(settings, info, ydl)
                self._apply_high_res_cover_art_mp3(settings, info, ydl)

            self._set_progress_threadsafe(100.0)
            self._set_status_threadsafe("Done")
            self.log("All tasks completed.")
        except yt_dlp.utils.DownloadCancelled:
            self._set_status_threadsafe("Cancelled")
            self.log("Download cancelled.")
        except Exception as exc:
            self._set_status_threadsafe("Failed")
            self.log(f"Error: {exc}")
            self.after(0, messagebox.showerror, "Download error", str(exc))
        finally:
            self.after(0, self._set_running_state, False)


def main() -> None:
    app = YouTubeDownloaderApp()
    app.mainloop()


if __name__ == "__main__":
    main()
