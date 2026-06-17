<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Công cụ Geotag & SEO Hình Ảnh Độc Lập</title>
    
    <!-- Tailwind CSS CDN -->
    <script src="https://cdn.tailwindcss.com"></script>
    <script>
        tailwind.config = {
            theme: {
                extend: {
                    animation: {
                        'spin-slow': 'spin 12s linear infinite',
                        'bounce-short': 'bounceShort 0.5s ease-out'
                    },
                    keyframes: {
                        bounceShort: {
                            '0%, 100%': { transform: 'translateY(0)' },
                            '50%': { transform: 'translateY(-4px)' }
                        }
                    }
                }
            }
        }
    </script>
    
    <!-- JSZip CDN for batch download -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js"></script>
    
    <!-- React & ReactDOM (UMD Production) -->
    <script src="https://unpkg.com/react@18/umd/react.production.min.js" crossorigin></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js" crossorigin></script>
    
    <!-- Babel standalone for on-the-fly JSX compilation -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    
    <style>
        /* Custom thanh cuộn đẹp mắt */
        ::-webkit-scrollbar {
            width: 6px;
        }
        ::-webkit-scrollbar-track {
            background: #f1f5f9;
        }
        ::-webkit-scrollbar-thumb {
            background: #cbd5e1;
            border-radius: 4px;
        }
        ::-webkit-scrollbar-thumb:hover {
            background: #94a3b8;
        }
    </style>
</head>
<body class="bg-slate-50 text-slate-800 font-sans">
    
    <!-- React root mount point -->
    <div id="root"></div>

    <!-- React App Script -->
    <script type="text/babel">
        const { useState, useEffect, useRef, useMemo } = React;

        // --- CÁC THƯ VIỆN ĐỘC LẬP TÍCH HỢP INLINE ---
        const ACCEPTED_TYPES = ["image/jpeg", "image/png", "image/webp", "image/gif", "image/avif"];
        const MAX_SIZE = 50 * 1024 * 1024; // 50MB

        function clampNumber(value, min, max) {
            const n = Number(value);
            return Number.isFinite(n) && n >= min && n <= max;
        }

        // Chuyển chuỗi thành định dạng chuẩn SEO (slug)
        function slugify(text) {
            if (!text) return "";
            return text.toString()
                .normalize("NFD")
                .replace(/[\u0300-\u036f]/g, "") // Xóa dấu tiếng Việt
                .replace(/đ/g, "d").replace(/Đ/g, "D")
                .replace(/[^a-zA-Z0-9\s-_]/g, "") // Chỉ giữ chữ, số, khoảng trắng, gạch ngang, gạch dưới
                .trim()
                .replace(/\s+/g, "-") // Thay khoảng trắng bằng dấu gạch ngang
                .replace(/-+/g, "-") // Loại bỏ gạch ngang thừa
                .toLowerCase();
        }

        // Xử lý tạo tên ảnh: Tiêu đề + Tên ảnh cũ
        function getOutputFilename(originalName, title, ext) {
            let oldName = originalName.replace(/\.[^.]+$/, "");
            
            const slugTitle = slugify(title);
            const slugOldName = slugify(oldName) || oldName.replace(/[^a-zA-Z0-9-_]/g, "").trim();

            if (slugTitle) {
                return `${slugTitle}-${slugOldName}.${ext}`;
            }
            return `${slugOldName}-geotag.${ext}`;
        }

        function readFileAsDataURL(file) {
            return new Promise((resolve, reject) => {
                const reader = new FileReader();
                reader.onload = () => resolve(reader.result);
                reader.onerror = reject;
                reader.readAsDataURL(file);
            });
        }

        function loadImage(src) {
            return new Promise((resolve, reject) => {
                const img = new Image();
                img.onload = () => resolve(img);
                img.onerror = reject;
                img.src = src;
            });
        }

        function dataURLToUint8Array(dataURL) {
            const base64 = dataURL.split(",")[1];
            const binary = atob(base64);
            const bytes = new Uint8Array(binary.length);
            for (let i = 0; i < binary.length; i++) bytes[i] = binary.charCodeAt(i);
            return bytes;
        }

        function uint8ArrayToBlob(bytes, type) {
            return new Blob([bytes], { type });
        }

        function encodeUtf16Le(str = "") {
            const bytes = new Uint8Array(str.length * 2 + 2);
            for (let i = 0; i < str.length; i++) {
                const code = str.charCodeAt(i);
                bytes[i * 2] = code & 0xff;
                bytes[i * 2 + 1] = (code >> 8) & 0xff;
            }
            return bytes;
        }

        function encodeUtf8(str = "") {
            return new TextEncoder().encode(String(str));
        }

        function encodeUtf8WithNull(str = "") {
            const utf8 = encodeUtf8(str);
            const bytes = new Uint8Array(utf8.length + 1);
            bytes.set(utf8);
            bytes[utf8.length] = 0;
            return bytes;
        }

        // Ghép nối mảng byte
        function concatBytes(...arrays) {
            const total = arrays.reduce((sum, arr) => sum + arr.length, 0);
            const out = new Uint8Array(total);
            let offset = 0;
            arrays.forEach((arr) => {
                out.set(arr, offset);
                offset += arr.length;
            });
            return out;
        }

        function writeU16BE(value) {
            return new Uint8Array([(value >> 8) & 255, value & 255]);
        }

        function writeU32BE(value) {
            return new Uint8Array([(value >>> 24) & 255, (value >>> 16) & 255, (value >>> 8) & 255, value & 255]);
        }

        function makeRational(value, scale = 1000000) {
            const numerator = Math.round(Math.abs(value) * scale);
            return [numerator, scale];
        }

        function decimalToDmsRationals(decimal) {
            const absolute = Math.abs(Number(decimal));
            const degrees = Math.floor(absolute);
            const minutesFloat = (absolute - degrees) * 60;
            const minutes = Math.floor(minutesFloat);
            const seconds = (minutesFloat - minutes) * 60;
            return [[degrees, 1], [minutes, 1], makeRational(seconds, 1000000)];
        }

        function entry(tag, type, count, data) {
            return { tag, type, count, data };
        }

        function asciiEntry(tag, str) {
            const bytes = encodeUtf8WithNull(str);
            return entry(tag, 2, bytes.length, bytes);
        }

        function byteEntry(tag, bytes) {
            return entry(tag, 1, bytes.length, bytes);
        }

        function shortEntry(tag, value) {
            const bytes = new Uint8Array(2);
            const view = new DataView(bytes.buffer);
            view.setUint16(0, value, true);
            return entry(tag, 3, 1, bytes);
        }

        function longEntry(tag, value) {
            const bytes = new Uint8Array(4);
            const view = new DataView(bytes.buffer);
            view.setUint32(0, value, true);
            return entry(tag, 4, 1, bytes);
        }

        function rationalEntry(tag, values) {
            const bytes = new Uint8Array(values.length * 8);
            const view = new DataView(bytes.buffer);
            values.forEach(([num, den], i) => {
                view.setUint32(i * 8, num, true);
                view.setUint32(i * 8 + 4, den, true);
            });
            return entry(tag, 5, values.length, bytes);
        }

        function compileTiff(ifd0Entries, gpsEntries, exifEntries) {
            ifd0Entries.sort((a, b) => a.tag - b.tag);
            gpsEntries.sort((a, b) => a.tag - b.tag);
            exifEntries.sort((a, b) => a.tag - b.tag);

            const tiffHeaderSize = 8;
            const ifd0Size = 2 + ifd0Entries.length * 12 + 4;
            const gpsSize = gpsEntries.length > 0 ? (2 + gpsEntries.length * 12 + 4) : 0;
            const exifSize = exifEntries.length > 0 ? (2 + exifEntries.length * 12 + 4) : 0;

            const gpsOffset = tiffHeaderSize + ifd0Size;
            const exifOffset = gpsOffset + gpsSize;
            const dataStartOffset = exifOffset + exifSize;

            ifd0Entries.forEach(e => {
                if (e.tag === 0x8769 && exifSize > 0) {
                    new DataView(e.data.buffer).setUint32(0, exifOffset, true);
                }
                if (e.tag === 0x8825 && gpsSize > 0) {
                    new DataView(e.data.buffer).setUint32(0, gpsOffset, true);
                }
            });

            let currentDataOffset = dataStartOffset;
            const trailingDataBlocks = [];

            function processEntries(entries) {
                const buffer = new Uint8Array(2 + entries.length * 12 + 4);
                const view = new DataView(buffer.buffer);
                view.setUint16(0, entries.length, true);

                entries.forEach((e, idx) => {
                    const base = 2 + idx * 12;
                    view.setUint16(base, e.tag, true);
                    view.setUint16(base + 2, e.type, true);
                    view.setUint32(base + 4, e.count, true);

                    const dataLen = e.data.length;
                    if (dataLen <= 4) {
                        const temp = new Uint8Array(4);
                        temp.set(e.data);
                        buffer.set(temp, base + 8);
                    } else {
                        view.setUint32(base + 8, currentDataOffset, true);
                        trailingDataBlocks.push(e.data);
                        currentDataOffset += dataLen;
                        if (dataLen % 2 !== 0) {
                            trailingDataBlocks.push(new Uint8Array([0]));
                            currentDataOffset += 1;
                        }
                    }
                });

                view.setUint32(buffer.length - 4, 0, true);
                return buffer;
            }

            const ifd0Block = processEntries(ifd0Entries);
            const gpsBlock = gpsSize > 0 ? processEntries(gpsEntries) : new Uint8Array(0);
            const exifBlock = exifSize > 0 ? processEntries(exifEntries) : new Uint8Array(0);

            const tiffHeader = new Uint8Array(8);
            const headerView = new DataView(tiffHeader.buffer);
            tiffHeader[0] = 0x49;
            tiffHeader[1] = 0x49;
            headerView.setUint16(2, 42, true);
            headerView.setUint32(4, 8, true);

            return concatBytes(tiffHeader, ifd0Block, gpsBlock, exifBlock, ...trailingDataBlocks);
        }

        function encodeUserComment(str = "") {
            const prefix = encodeUtf8("UNICODE\0");
            const utf16 = new Uint8Array(str.length * 2);
            for (let i = 0; i < str.length; i++) {
                const code = str.charCodeAt(i);
                utf16[i * 2] = code & 0xff;
                utf16[i * 2 + 1] = (code >> 8) & 0xff;
            }
            return concatBytes(prefix, utf16);
        }

        function buildExif({ lat, lng, title, subject, keywords, copyright, author, date, comment, rating }) {
            const ifd0Entries = [];
            const gpsEntries = [];
            const exifEntries = [];

            ifd0Entries.push(longEntry(0x8769, 0));
            ifd0Entries.push(longEntry(0x8825, 0));

            if (title) {
                ifd0Entries.push(asciiEntry(0x010e, title));
                ifd0Entries.push(byteEntry(0x9c9b, encodeUtf16Le(title)));
            }
            if (author) {
                ifd0Entries.push(asciiEntry(0x013b, author));
                ifd0Entries.push(byteEntry(0x9c9d, encodeUtf16Le(author)));
            }
            if (copyright) {
                ifd0Entries.push(asciiEntry(0x8298, copyright));
            }
            if (rating) {
                ifd0Entries.push(shortEntry(0x4746, Number(rating)));
            }
            if (subject) {
                ifd0Entries.push(byteEntry(0x9c9f, encodeUtf16Le(subject)));
            }
            if (keywords) {
                ifd0Entries.push(byteEntry(0x9c9e, encodeUtf16Le(keywords)));
            }
            if (comment) {
                ifd0Entries.push(byteEntry(0x9c9c, encodeUtf16Le(comment)));
            }

            const latDms = decimalToDmsRationals(lat);
            const lngDms = decimalToDmsRationals(lng);
            gpsEntries.push(byteEntry(0x0000, new Uint8Array([2, 3, 0, 0])));
            gpsEntries.push(asciiEntry(0x0001, Number(lat) >= 0 ? "N" : "S"));
            gpsEntries.push(rationalEntry(0x0002, latDms));
            gpsEntries.push(asciiEntry(0x0003, Number(lng) >= 0 ? "E" : "W"));
            gpsEntries.push(rationalEntry(0x0004, lngDms));
            
            if (date) {
                gpsEntries.push(asciiEntry(0x001d, String(date).replace(/-/g, ":")));
            }

            const commentParts = [title, subject, keywords, comment].filter(Boolean).join(" | ");
            if (commentParts) {
                exifEntries.push(byteEntry(0x9286, encodeUserComment(commentParts)));
            }
            if (date) {
                const exifDate = String(date).replace(/-/g, ":") + " 12:00:00";
                exifEntries.push(asciiEntry(0x9003, exifDate));
            }

            const tiffBytes = compileTiff(ifd0Entries, gpsEntries, exifEntries);
            const exifHeader = encodeUtf8("Exif\0\0");
            const payload = concatBytes(exifHeader, tiffBytes);
            const length = payload.length + 2;
            const jpegSegment = concatBytes(new Uint8Array([0xff, 0xe1]), writeU16BE(length), payload);

            return { jpegSegment, tiffBytes };
        }

        function insertExifIntoJpeg(jpegBytes, exifSegment) {
            if (jpegBytes[0] !== 0xff || jpegBytes[1] !== 0xd8) throw new Error("Không phải JPEG hợp lệ");
            let offset = 2;
            while (offset + 4 < jpegBytes.length && jpegBytes[offset] === 0xff) {
                const marker = jpegBytes[offset + 1];
                if (marker < 0xe0 || marker > 0xef) break;
                const size = (jpegBytes[offset + 2] << 8) + jpegBytes[offset + 3];
                offset += 2 + size;
            }
            return concatBytes(jpegBytes.slice(0, 2), exifSegment, jpegBytes.slice(offset));
        }

        function parseWebP(bytes) {
            if (bytes[0] !== 0x52 || bytes[1] !== 0x49 || bytes[2] !== 0x46 || bytes[3] !== 0x46) {
                throw new Error("Không phải định dạng RIFF hợp lệ");
            }
            if (bytes[8] !== 0x57 || bytes[9] !== 0x45 || bytes[10] !== 0x42 || bytes[11] !== 0x50) {
                throw new Error("Không phải định dạng WEBP hợp lệ");
            }

            let width = 0;
            let height = 0;
            let hasAlpha = false;
            let hasICC = false;

            let offset = 12;
            const imageChunks = [];
            const otherChunks = [];

            while (offset < bytes.length) {
                if (offset + 8 > bytes.length) break;
                const tag = String.fromCharCode(bytes[offset], bytes[offset + 1], bytes[offset + 2], bytes[offset + 3]);
                const size = bytes[offset + 4] | (bytes[offset + 5] << 8) | (bytes[offset + 6] << 16) | (bytes[offset + 7] << 24);
                if (offset + 8 + size > bytes.length) break;

                const chunkData = bytes.slice(offset + 8, offset + 8 + size);

                if (tag === "VP8X") {
                    const flags = chunkData[0];
                    hasICC = (flags & 0x20) !== 0;
                    hasAlpha = (flags & 0x10) !== 0;
                    width = (chunkData[4] | (chunkData[5] << 8) | (chunkData[6] << 16)) + 1;
                    height = (chunkData[7] | (chunkData[8] << 8) | (chunkData[9] << 16)) + 1;
                } else if (tag === "VP8 ") {
                    if (width === 0) {
                        if (chunkData[3] === 0x9d && chunkData[4] === 0x01 && chunkData[5] === 0x2a) {
                            width = (chunkData[6] | (chunkData[7] << 8)) & 0x3fff;
                            height = (chunkData[8] | (chunkData[9] << 8)) & 0x3fff;
                        }
                    }
                    imageChunks.push({ tag, data: chunkData });
                } else if (tag === "VP8L") {
                    if (width === 0) {
                        if (chunkData[0] === 0x2f) {
                            const val = chunkData[1] | (chunkData[2] << 8) | (chunkData[3] << 16) | (chunkData[4] << 24);
                            width = (val & 0x3fff) + 1;
                            height = ((val >> 14) & 0x3fff) + 1;
                            hasAlpha = ((val >> 28) & 1) === 1;
                        }
                    }
                    imageChunks.push({ tag, data: chunkData });
                } else if (tag === "EXIF" || tag === "XMP") {
                    // Loại bỏ khối cũ
                } else {
                    otherChunks.push({ tag, data: chunkData });
                }

                offset += 8 + size;
                if (size % 2 !== 0) offset++;
            }

            return { width, height, hasAlpha, hasICC, imageChunks, otherChunks };
        }

        function writeWebPChunk(tag, data) {
            const tagBytes = encodeUtf8(tag).slice(0, 4);
            const size = data.length;
            const sizeBytes = new Uint8Array([
                size & 255,
                (size >> 8) & 255,
                (size >> 16) & 255,
                (size >> 24) & 255
            ]);
            const padding = size % 2 !== 0 ? new Uint8Array([0]) : new Uint8Array([]);
            return concatBytes(tagBytes, sizeBytes, data, padding);
        }

        function insertExifIntoWebp(webpBytes, tiffBytes, fallbackWidth, fallbackHeight) {
            const parsed = parseWebP(webpBytes);
            const width = parsed.width || fallbackWidth || 800;
            const height = parsed.height || fallbackHeight || 600;

            const vp8xPayload = new Uint8Array(10);
            let flags = 0x08;
            if (parsed.hasICC) flags |= 0x20;
            if (parsed.hasAlpha) flags |= 0x10;
            vp8xPayload[0] = flags;

            const wMinusOne = width - 1;
            const hMinusOne = height - 1;

            vp8xPayload[4] = wMinusOne & 255;
            vp8xPayload[5] = (wMinusOne >> 8) & 255;
            vp8xPayload[6] = (wMinusOne >> 16) & 255;

            vp8xPayload[7] = hMinusOne & 255;
            vp8xPayload[8] = (hMinusOne >> 8) & 255;
            vp8xPayload[9] = (hMinusOne >> 16) & 255;

            const vp8xChunk = writeWebPChunk("VP8X", vp8xPayload);

            const middleChunks = [];
            parsed.otherChunks.forEach(c => {
                if (c.tag === "ICCP") middleChunks.push(writeWebPChunk(c.tag, c.data));
            });
            parsed.imageChunks.forEach(c => {
                middleChunks.push(writeWebPChunk(c.tag, c.data));
            });
            parsed.otherChunks.forEach(c => {
                if (c.tag !== "ICCP") middleChunks.push(writeWebPChunk(c.tag, c.data));
            });

            const exifChunk = writeWebPChunk("EXIF", tiffBytes);
            const payload = concatBytes(vp8xChunk, ...middleChunks, exifChunk);

            const riffHeader = new Uint8Array(12);
            riffHeader.set(encodeUtf8("RIFF").slice(0, 4), 0);
            const totalSize = payload.length + 4;
            riffHeader[4] = totalSize & 255;
            riffHeader[5] = (totalSize >> 8) & 255;
            riffHeader[6] = (totalSize >> 16) & 255;
            riffHeader[7] = (totalSize >> 24) & 255;
            riffHeader.set(encodeUtf8("WEBP").slice(0, 4), 8);

            return concatBytes(riffHeader, payload);
        }

        function crc32(bytes) {
            let crc = -1;
            for (let i = 0; i < bytes.length; i++) {
                crc ^= bytes[i];
                for (let j = 0; j < 8; j++) crc = (crc >>> 1) ^ (0xedb88320 & -(crc & 1));
            }
            return (crc ^ -1) >>> 0;
        }

        function makePngiTXtChunk(keyword, text) {
            const type = encodeUtf8("iTXt");
            const keywordBytes = encodeUtf8(keyword.slice(0, 79)); 
            const textBytes = encodeUtf8(String(text || ""));

            const data = concatBytes(
                keywordBytes,
                new Uint8Array([0]),
                new Uint8Array([0]),
                new Uint8Array([0]),
                new Uint8Array([0]),
                new Uint8Array([0]),
                textBytes
            );

            const crc = crc32(concatBytes(type, data));
            return concatBytes(writeU32BE(data.length), type, data, writeU32BE(crc));
        }

        function insertTextChunksIntoPng(pngBytes, metadata) {
            const signature = pngBytes.slice(0, 8);
            const chunks = Object.entries(metadata)
                .filter(([, value]) => value !== undefined && value !== null && String(value).trim() !== "")
                .map(([key, value]) => makePngiTXtChunk(key, value));
            return concatBytes(signature, pngBytes.slice(8, 33), ...chunks, pngBytes.slice(33));
        }

        async function renderImage(file, outputFormat, metadata) {
            const src = await readFileAsDataURL(file);
            const img = await loadImage(src);
            const canvas = document.createElement("canvas");
            const width = img.naturalWidth || img.width;
            const height = img.naturalHeight || img.height;
            canvas.width = width;
            canvas.height = height;
            
            const ctx = canvas.getContext("2d");
            if (outputFormat === "jpg") {
                ctx.fillStyle = "#fff";
                ctx.fillRect(0, 0, canvas.width, canvas.height);
            }
            ctx.drawImage(img, 0, 0);

            let mime = "image/jpeg";
            if (outputFormat === "png") mime = "image/png";
            else if (outputFormat === "webp") mime = "image/webp";

            const dataURL = canvas.toDataURL(mime, 0.94);
            let bytes = dataURLToUint8Array(dataURL);

            const { jpegSegment, tiffBytes } = buildExif(metadata);

            if (outputFormat === "jpg") {
                bytes = insertExifIntoJpeg(bytes, jpegSegment);
            } else if (outputFormat === "webp") {
                bytes = insertExifIntoWebp(bytes, tiffBytes, width, height);
            } else {
                bytes = insertTextChunksIntoPng(bytes, {
                    GPSLatitude: metadata.lat,
                    GPSLongitude: metadata.lng,
                    Title: metadata.title,
                    Subject: metadata.subject,
                    Keywords: metadata.keywords,
                    Copyright: metadata.copyright,
                    Author: metadata.author,
                    Date: metadata.date,
                    Comment: metadata.comment,
                    Rating: metadata.rating,
                });
            }

            return uint8ArrayToBlob(bytes, mime);
        }

        function downloadBlob(blob, filename) {
            const url = URL.createObjectURL(blob);
            const a = document.createElement("a");
            a.href = url;
            a.download = filename;
            document.body.appendChild(a);
            a.click();
            a.remove();
            URL.revokeObjectURL(url);
        }

        // --- CÁC THÀNH PHẦN ICON SVG INLINE SANG TRỌNG ---
        const SVG = {
            Globe: () => (
                <svg className="w-7 h-7" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth="2.5">
                    <circle cx="12" cy="12" r="10" />
                    <path strokeLinecap="round" strokeLinejoin="round" d="M12 2a14.5 14.5 0 000 20M2 12h20" />
                </svg>
            ),
            Upload: () => (
                <svg className="w-10 h-10 mx-auto mb-3 text-emerald-500 animate-pulse" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth="2">
                    <path strokeLinecap="round" strokeLinejoin="round" d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-8l-4-4m0 0L8 8m4-4v12" />
                </svg>
            ),
            Image: () => (
                <svg className="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth="2">
                    <rect x="3" y="3" width="18" height="18" rx="2" ry="2" />
                    <circle cx="8.5" cy="8.5" r="1.5" />
                    <polyline points="21 15 16 10 5 21" />
                </svg>
            ),
            Trash: () => (
                <svg className="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth="2">
                    <polyline points="3 6 5 6 21 6" />
                    <path d="M19 6v14a2 2 0 01-2 2H7a2 2 0 01-2-2V6m3 0V4a2 2 0 012-2h4a2 2 0 012 2v2" />
                </svg>
            ),
            Settings: () => (
                <svg className="w-5 h-5 text-indigo-500" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth="2">
                    <circle cx="12" cy="12" r="3" />
                    <path d="M19.4 15a1.65 1.65 0 00.33 1.82l.06.06a2 2 0 11-2.83 2.83l-.06-.06a1.65 1.65 0 00-1.82-.33 1.65 1.65 0 00-1 1.51V21a2 2 0 01-4 0v-.09A1.65 1.65 0 009 19.4a1.65 1.65 0 00-1.82.33l-.06.06a2 2 0 11-2.83-2.83l.06-.06a1.65 1.65 0 00.33-1.82 1.65 1.65 0 00-1.51-1H3a2 2 0 010-4h.09A1.65 1.65 0 004.6 9a1.65 1.65 0 00-.33-1.82l-.06-.06a2 2 0 112.83-2.83l.06.06a1.65 1.65 0 001.82.33H9a1.65 1.65 0 001-1.51V3a2 2 0 014 0v.09a1.65 1.65 0 001 1.51 1.65 1.65 0 001.82-.33l.06-.06a2 2 0 112.83 2.83l-.06.06a1.65 1.65 0 00-.33 1.82V9a1.65 1.65 0 001.51 1H21a2 2 0 010 4h-.09a1.65 1.65 0 00-1.51 1z" />
                </svg>
            ),
            Map: () => (
                <svg className="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth="2">
                    <polygon points="1 6 1 22 8 18 16 22 23 18 23 2 16 6 8 2 1 6" />
                    <line x1="8" y1="2" x2="8" y2="18" />
                    <line x1="16" y1="6" x2="16" y2="22" />
                </svg>
            ),
            Sparkles: () => (
                <svg className="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth="2">
                    <path d="M12 3v1m0 16v1m9-9h-1M4 12H3m15.364-6.364l-.707.707M6.343 17.657l-.707.707m0-12.728l.707.707m11.314 11.314l.707.707" />
                </svg>
            ),
            Locate: () => (
                <svg className="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth="2">
                    <circle cx="12" cy="12" r="10" />
                    <circle cx="12" cy="12" r="3" />
                    <line x1="12" y1="1" x2="12" y2="3" />
                    <line x1="12" y1="21" x2="12" y2="23" />
                    <line x1="1" y1="12" x2="3" y2="12" />
                    <line x1="21" y1="12" x2="23" y2="12" />
                </svg>
            ),
            Info: () => (
                <svg className="w-4 h-4 text-indigo-500 shrink-0 mt-0.5" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth="2">
                    <circle cx="12" cy="12" r="10" />
                    <line x1="12" y1="16" x2="12" y2="12" />
                    <line x1="12" y1="8" x2="12.01" y2="8" />
                </svg>
            )
        };

        // --- COMPONENT CHÍNH ---
        function GeotagImageTool() {
            const inputRef = useRef(null);

            const [files, setFiles] = useState([]);
            const [dragging, setDragging] = useState(false);
            const [lat, setLat] = useState("21.028511"); // Mặc định Hà Nội
            const [lng, setLng] = useState("105.854223");
            
            const [activeTab, setActiveTab] = useState("location");
            const [outputFormat, setOutputFormat] = useState("jpg");
            
            const [metadata, setMetadata] = useState({
                title: "",
                subject: "",
                keywords: "",
                copyright: "",
                author: "",
                date: new Date().toISOString().split("T")[0],
                comment: "",
                rating: 5,
            });

            const [processing, setProcessing] = useState(false);
            const [message, setMessage] = useState({ type: "", text: "" });

            const previews = useMemo(() => files.map((file) => ({ file, url: URL.createObjectURL(file) })), [files]);
            const canSubmit = files.length > 0 && clampNumber(lat, -90, 90) && clampNumber(lng, -180, 180) && !processing;

            // Xử lý Dynamic Text cho nút bấm chính giúp người dùng luôn thấy và hiểu trạng thái
            const submitButtonText = useMemo(() => {
                if (processing) return "Đang xử lý cấu trúc ảnh...";
                if (files.length === 0) return "Vui lòng kéo/tải ảnh lên trước";
                if (!clampNumber(lat, -90, 90) || !clampNumber(lng, -180, 180)) return "Tọa độ GPS chưa hợp lệ";
                
                return files.length > 1 ? "Ghi đè & Tải về gói ZIP" : "Ghi đè & Tải về ảnh";
            }, [processing, files.length, lat, lng]);

            function showToast(type, text) {
                setMessage({ type, text });
                setTimeout(() => {
                    setMessage((current) => current.text === text ? { type: "", text: "" } : current);
                }, 6000);
            }

            function addFiles(fileList) {
                const picked = Array.from(fileList || []);
                const valid = picked.filter((file) => ACCEPTED_TYPES.includes(file.type) && file.size <= MAX_SIZE);
                if (valid.length !== picked.length) {
                    showToast("error", "Một số file đã bị bỏ qua vì sai định dạng hoặc kích thước lớn hơn 50MB.");
                }
                setFiles((current) => [...current, ...valid]);
                if (valid.length > 0) {
                    showToast("success", `Đã nạp thành công ${valid.length} hình ảnh.`);
                }
            }

            // Gỡ bỏ tệp tin
            function removeFile(index) {
                setFiles((current) => current.filter((_, i) => i !== index));
            }

            function clearAllFiles() {
                setFiles([]);
                showToast("info", "Đã dọn dẹp danh sách hàng chờ.");
            }

            function useCurrentLocation() {
                if (!navigator.geolocation) {
                    showToast("error", "Thiết bị hoặc trình duyệt không hỗ trợ dịch vụ vị trí.");
                    return;
                }
                navigator.geolocation.getCurrentPosition(
                    (pos) => {
                        setLat(pos.coords.latitude.toFixed(7));
                        setLng(pos.coords.longitude.toFixed(7));
                        showToast("success", "Đã định vị thành công vị trí hiện tại.");
                    },
                    () => showToast("error", "Không thể lấy vị trí hiện tại. Hãy kiểm tra lại cấp quyền GPS.")
                );
            }

            async function handleSubmit() {
                if (files.length === 0) return showToast("error", "Vui lòng chọn ít nhất một hình ảnh.");
                if (!clampNumber(lat, -90, 90)) return showToast("error", "Vĩ độ nằm ngoài phạm vi hoạt động (-90 đến 90).");
                if (!clampNumber(lng, -180, 180)) return showToast("error", "Kinh độ nằm ngoài phạm vi hoạt động (-180 đến 180).");

                setProcessing(true);
                showToast("info", "Bắt đầu ghi đè Geotag & SEO metadata...");

                try {
                    const meta = { ...metadata, lat: Number(lat), lng: Number(lng) };
                    const ext = outputFormat;
                    const processed = [];

                    for (const file of files) {
                        const blob = await renderImage(file, outputFormat, meta);
                        const outFilename = getOutputFilename(file.name, meta.title, ext);
                        processed.push({ blob, filename: outFilename });
                    }

                    if (processed.length === 1) {
                        downloadBlob(processed[0].blob, processed[0].filename);
                        showToast("success", "Đã lưu thành công metadata và tải ảnh xuống!");
                    } else {
                        if (!window.JSZip) {
                            throw new Error("Không thể liên kết thư viện lưu trữ JSZip.");
                        }
                        const zip = new window.JSZip();
                        processed.forEach(({ blob, filename }) => zip.file(filename, blob));
                        const zipBlob = await zip.generateAsync({ type: "blob" });
                        downloadBlob(zipBlob, `geotagged-images-${Date.now()}.zip`);
                        showToast("success", `Hoàn tất! Đã nén và tải về toàn bộ ${processed.length} hình ảnh.`);
                    }
                } catch (error) {
                    console.error(error);
                    showToast("error", "Có lỗi xảy ra khi đóng gói tệp tin nhị phân. Hãy kiểm tra định dạng hình ảnh nạp vào.");
                } finally {
                    setProcessing(false);
                }
            }

            return (
                <div className="min-h-screen bg-slate-50 text-slate-800 font-sans">
                    {/* Thông báo nổi (Toast Notification) */}
                    {message.text && (
                        <div className="fixed top-5 right-5 z-[9999] max-w-md animate-bounce-short rounded-xl border border-slate-100 bg-white p-4 shadow-xl transition-all">
                            <div className="flex items-start gap-3">
                                <div className="text-emerald-500 shrink-0">
                                    <svg className="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth="2.5">
                                        <path strokeLinecap="round" strokeLinejoin="round" d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z" />
                                    </svg>
                                </div>
                                <div>
                                    <p className="text-sm font-semibold text-slate-900">Thông báo</p>
                                    <p className="mt-0.5 text-xs text-slate-600 leading-relaxed">{message.text}</p>
                                </div>
                            </div>
                        </div>
                    )}

                    {/* Header */}
                    <header className="relative border-b bg-white overflow-hidden shadow-sm">
                        <div className="absolute inset-0 bg-gradient-to-r from-emerald-500/10 via-transparent to-indigo-500/10" />
                        <div className="relative mx-auto max-w-7xl px-4 py-8 sm:px-6 lg:px-8">
                            <div className="flex flex-col md:flex-row md:items-center justify-between gap-6">
                                <div className="flex items-center gap-4">
                                    <div className="flex items-center justify-center rounded-2xl bg-gradient-to-tr from-emerald-500 to-teal-600 p-3.5 text-white shadow-lg shadow-emerald-500/20">
                                        <SVG.Globe />
                                    </div>
                                    <div>
                                        <h1 className="text-2xl sm:text-3xl font-extrabold text-slate-900 tracking-tight">Geotag & SEO Metadata Tool</h1>
                                        <p className="mt-1 text-sm text-slate-500">
                                            Nhúng tọa độ định vị GPS, thông tin thương hiệu và Metadata nâng cao hỗ trợ 100% tiếng Việt không lỗi vào ảnh JPG, PNG hoặc WEBP.
                                        </p>
                                    </div>
                                </div>
                                <div className="flex flex-wrap gap-2">
                                    <span className="inline-flex items-center gap-1.5 rounded-full bg-emerald-50 px-3 py-1 text-xs font-semibold text-emerald-700 border border-emerald-100">
                                        <span className="h-1.5 w-1.5 rounded-full bg-emerald-500 animate-ping" />
                                        Chạy độc lập (Bảo mật 100% Client-side)
                                    </span>
                                </div>
                            </div>
                        </div>
                    </header>

                    {/* Main Container */}
                    <main className="mx-auto max-w-7xl px-4 py-8 sm:px-6 lg:px-8">
                        <div className="grid grid-cols-1 lg:grid-cols-12 gap-8 items-start">
                            
                            {/* CỘT TRÁI - Nhập & Quản lý Hình ảnh */}
                            <section className="lg:col-span-5 space-y-6">
                                <div className="rounded-2xl border border-slate-200 bg-white p-6 shadow-sm">
                                    <div className="mb-4 flex items-center justify-between">
                                        <div className="flex items-center gap-2 font-bold text-slate-800">
                                            <SVG.Image />
                                            <span>Hình Ảnh Đầu Vào ({files.length})</span>
                                        </div>
                                        {files.length > 0 && (
                                            <button 
                                                onClick={clearAllFiles}
                                                className="flex items-center gap-1 text-xs font-medium text-rose-500 hover:text-rose-700 transition"
                                            >
                                                <SVG.Trash />
                                                Xóa tất cả
                                            </button>
                                        )}
                                    </div>

                                    {/* Khối kéo thả */}
                                    <div
                                        onClick={() => inputRef.current?.click()}
                                        onDragOver={(e) => {
                                            e.preventDefault();
                                            setDragging(true);
                                        }}
                                        onDragLeave={() => setDragging(false)}
                                        onDrop={(e) => {
                                            e.preventDefault();
                                            setDragging(false);
                                            addFiles(e.dataTransfer.files);
                                        }}
                                        className={`cursor-pointer rounded-2xl border-2 border-dashed p-8 text-center transition-all ${
                                            dragging 
                                                ? "border-emerald-500 bg-emerald-50/50 scale-[0.99] shadow-inner" 
                                                : "border-slate-300 hover:border-emerald-400 hover:bg-slate-50/50"
                                        }`}
                                    >
                                        <SVG.Upload />
                                        <p className="text-sm font-semibold text-slate-800">Kéo & thả ảnh của bạn vào đây</p>
                                        <p className="mt-1 text-xs text-slate-400">hoặc nhấp để chọn tệp từ thiết bị</p>
                                        <div className="mt-4 inline-flex items-center gap-2 rounded-lg bg-slate-100 px-2.5 py-1 text-[10px] font-medium text-slate-500">
                                            Định dạng: JPEG, PNG, WEBP, AVIF, GIF • Max 50MB
                                        </div>
                                        <input 
                                            ref={inputRef} 
                                            type="file" 
                                            accept="image/png,image/gif,image/jpeg,image/webp,image/avif" 
                                            multiple 
                                            className="hidden" 
                                            onChange={(e) => addFiles(e.target.files)} 
                                        />
                                    </div>

                                    {/* Danh sách ảnh đã nạp */}
                                    {previews.length > 0 ? (
                                        <div className="mt-6 space-y-3 max-h-[380px] overflow-y-auto pr-1">
                                            <p className="text-xs font-semibold text-slate-400 uppercase tracking-wider">Danh sách hình ảnh tải lên</p>
                                            {previews.map(({ file, url }, index) => (
                                                <div 
                                                    key={`${file.name}-${index}`} 
                                                    className="group flex items-center gap-3 rounded-xl border border-slate-150 bg-slate-50/50 p-2.5 transition hover:bg-slate-50"
                                                >
                                                    <div className="relative h-14 w-14 shrink-0 overflow-hidden rounded-lg border bg-white shadow-sm">
                                                        <img src={url} alt={file.name} className="h-full w-full object-cover" />
                                                    </div>
                                                    <div className="min-w-0 flex-1">
                                                        <p className="truncate text-xs font-bold text-slate-700">{file.name}</p>
                                                        <p className="mt-0.5 text-[10px] text-slate-400 font-mono">
                                                            {(file.size / (1024 * 1024)).toFixed(2)} MB
                                                        </p>
                                                    </div>
                                                    <button 
                                                        onClick={() => removeFile(index)} 
                                                        className="rounded-lg bg-white p-1.5 text-slate-400 border border-slate-200 shadow-sm transition hover:bg-rose-50 hover:text-rose-600 hover:border-rose-100"
                                                        aria-label="Xóa ảnh"
                                                    >
                                                        <svg className="w-3.5 h-3.5" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth="2.5"><path strokeLinecap="round" strokeLinejoin="round" d="M6 18L18 6M6 6l12 12" /></svg>
                                                    </button>
                                                </div>
                                            ))}
                                        </div>
                                    ) : (
                                        <div className="mt-6 flex flex-col items-center justify-center py-8 text-center border border-dashed rounded-xl bg-slate-50/40 text-slate-400">
                                            <SVG.Image />
                                            <p className="text-xs">Chưa có ảnh nào được chọn</p>
                                        </div>
                                    )}
                                </div>

                                {/* Định dạng đầu ra */}
                                <div className="rounded-2xl border border-slate-200 bg-white p-6 shadow-sm">
                                    <div className="mb-4 flex items-center gap-2 font-bold text-slate-800">
                                        <SVG.Settings />
                                        <span>Cấu Hình Định Dạng Xuất</span>
                                    </div>

                                    <div className="space-y-4">
                                        <div>
                                            <label className="text-xs font-bold text-slate-500 block mb-2">ĐỊNH DẠNG ĐẦU RA</label>
                                            <div className="grid grid-cols-3 gap-2 bg-slate-100 p-1 rounded-xl">
                                                {[
                                                    ["jpg", "JPG"],
                                                    ["png", "PNG"],
                                                    ["webp", "WEBP"],
                                                ].map(([value, label]) => (
                                                    <button
                                                        key={value}
                                                        onClick={() => setOutputFormat(value)}
                                                        className={`rounded-lg py-2.5 text-xs font-bold transition-all ${
                                                            outputFormat === value 
                                                                ? "bg-white text-slate-950 shadow-sm scale-100 font-extrabold" 
                                                                : "text-slate-500 hover:text-slate-800"
                                                        }`}
                                                    >
                                                        {label}
                                                    </button>
                                                ))}
                                            </div>
                                        </div>

                                        <div className="rounded-xl bg-slate-50 p-3 text-[11px] leading-relaxed text-slate-500 border border-slate-100">
                                            {outputFormat === "jpg" && (
                                                <div className="flex gap-2">
                                                    <SVG.Info />
                                                    <span>
                                                        <strong>Được đề xuất nhiều nhất:</strong> Định dạng JPG hỗ trợ 100% tiếng Việt có dấu nhờ cơ chế mã hóa nhị phân UTF-16LE, giúp tối ưu hóa xuất sắc các thuộc tính hiển thị trên File Explorer của Windows, Photoshop, Lightroom, macOS Finder.
                                                    </span>
                                                </div>
                                            )}
                                            {outputFormat === "png" && (
                                                <div className="flex gap-2">
                                                    <SVG.Info />
                                                    <span>
                                                        <strong>Ghi đè bằng chunk iTXt UTF-8:</strong> Đã được nâng cấp lên chuẩn `iTXt` hiện đại thay thế cho `tEXt` giới hạn cũ. Đảm bảo hỗ trợ hiển thị tiếng Việt có dấu nguyên vẹn khi phân tích siêu dữ liệu trên các công cụ tương thích.
                                                    </span>
                                                </div>
                                            )}
                                            {outputFormat === "webp" && (
                                                <div className="flex gap-2">
                                                    <SVG.Info />
                                                    <span>
                                                        <strong>WebP Extended (VP8X):</strong> Lưu ảnh WebP với cấu trúc nâng cấp để nhúng trực tiếp khối dữ liệu nhị phân <strong>EXIF GPS & Windows Details</strong> của JPG, duy trì dung lượng nén siêu nhẹ kèm SEO hoàn chỉnh không lỗi phông chữ!
                                                    </span>
                                                </div>
                                            )}
                                        </div>
                                    </div>
                                </div>
                            </section>

                            {/* CỘT PHẢI - Bản đồ, Tọa độ & Metadata SEO */}
                            <section className="lg:col-span-7 space-y-6">
                                
                                {/* Hệ thống TAB */}
                                <div className="flex bg-white rounded-2xl p-1.5 border border-slate-200 shadow-sm">
                                    {[
                                        { id: "location", label: "Tọa Độ Vị Trí", icon: SVG.Map },
                                        { id: "seo", label: "Metadata SEO Doanh Nghiệp", icon: SVG.Sparkles }
                                    ].map((tab) => {
                                        const Icon = tab.icon;
                                        return (
                                            <button
                                                key={tab.id}
                                                onClick={() => setActiveTab(tab.id)}
                                                className={`flex-1 flex items-center justify-center gap-2 py-3 rounded-xl text-xs sm:text-sm font-bold transition-all ${
                                                    activeTab === tab.id 
                                                        ? "bg-slate-950 text-white shadow-md" 
                                                        : "text-slate-500 hover:text-slate-800 hover:bg-slate-50"
                                                }`}
                                            >
                                                <Icon />
                                                <span>{tab.label}</span>
                                            </button>
                                        );
                                    })}
                                </div>

                                {/* TAB 1: BẢN ĐỒ & TỌA ĐỘ */}
                                {activeTab === "location" && (
                                    <div className="rounded-2xl border border-slate-200 bg-white p-6 shadow-sm space-y-6">
                                        <div className="flex flex-col sm:flex-row sm:items-center justify-between gap-3">
                                            <div className="flex items-center gap-2 font-bold text-slate-800">
                                                <svg className="w-5 h-5 text-emerald-500" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth="2"><path strokeLinecap="round" strokeLinejoin="round" d="M17.657 16.657L13.414 20.9a1.998 1.998 0 01-2.827 0l-4.244-4.243a8 8 0 1111.314 0z" /><path strokeLinecap="round" strokeLinejoin="round" d="M15 11a3 3 0 11-6 0 3 3 0 016 0z" /></svg>
                                                <span>Nhập Tọa Độ Địa Lý (GPS)</span>
                                            </div>
                                            <button 
                                                onClick={useCurrentLocation}
                                                className="inline-flex items-center justify-center gap-1.5 rounded-xl border border-slate-200 bg-white px-3 py-1.5 text-xs font-bold text-slate-700 shadow-sm transition hover:bg-slate-50"
                                            >
                                                <SVG.Locate />
                                                Sử dụng vị trí thiết bị
                                            </button>
                                        </div>

                                        {/* Trường hiển thị tọa độ điền tay */}
                                        <div className="grid gap-4 sm:grid-cols-2 pt-2">
                                            <label className="block text-xs font-bold text-slate-500 uppercase">
                                                <span className="mb-2 block">Vĩ độ (Latitude) *</span>
                                                <input 
                                                    value={lat} 
                                                    onChange={(e) => setLat(e.target.value)} 
                                                    type="number" 
                                                    step="any" 
                                                    placeholder="Ví dụ: 21.028511" 
                                                    className="w-full rounded-xl border border-slate-200 px-3.5 py-3 text-sm font-semibold text-slate-800 outline-none transition focus:border-emerald-500 focus:ring-2 focus:ring-emerald-500/20" 
                                                />
                                            </label>
                                            <label className="block text-xs font-bold text-slate-500 uppercase">
                                                <span className="mb-2 block">Kinh độ (Longitude) *</span>
                                                <input 
                                                    value={lng} 
                                                    onChange={(e) => setLng(e.target.value)} 
                                                    type="number" 
                                                    step="any" 
                                                    placeholder="Ví dụ: 105.854223" 
                                                    className="w-full rounded-xl border border-slate-200 px-3.5 py-3 text-sm font-semibold text-slate-800 outline-none transition focus:border-emerald-500 focus:ring-2 focus:ring-emerald-500/20" 
                                                />
                                            </label>
                                        </div>
                                    </div>
                                )}

                                {/* TAB 2: SEO & METADATA DOANH NGHIỆP */}
                                {activeTab === "seo" && (
                                    <div className="rounded-2xl border border-slate-200 bg-white p-6 shadow-sm space-y-4">
                                        <div className="mb-2 flex items-center gap-2 font-bold text-slate-800 border-b pb-3">
                                            <SVG.Sparkles />
                                            <span>Cấu Hình Metadata SEO Doanh Nghiệp (Không bắt buộc)</span>
                                        </div>

                                        <div className="grid gap-4 sm:grid-cols-2">
                                            <label className="block text-xs font-bold text-slate-500 uppercase">
                                                <span className="mb-1.5 block">Tiêu đề ảnh (Title) *</span>
                                                <input 
                                                    value={metadata.title} 
                                                    onChange={(e) => setMetadata((m) => ({ ...m, title: e.target.value }))} 
                                                    placeholder="Ví dụ: Tấm Ốp Than Tre Vân Sóng"
                                                    className="w-full rounded-xl border border-slate-200 px-3.5 py-2.5 text-xs outline-none transition focus:border-emerald-500 focus:ring-2 focus:ring-emerald-500/20" 
                                                />
                                            </label>

                                            <label className="block text-xs font-bold text-slate-500 uppercase">
                                                <span className="mb-1.5 block">Chủ đề (Subject)</span>
                                                <input 
                                                    value={metadata.subject} 
                                                    onChange={(e) => setMetadata((m) => ({ ...m, subject: e.target.value }))} 
                                                    placeholder="Ví dụ: Trang trí nội thất cao cấp"
                                                    className="w-full rounded-xl border border-slate-200 px-3.5 py-2.5 text-xs outline-none transition focus:border-emerald-500 focus:ring-2 focus:ring-emerald-500/20" 
                                                />
                                            </label>

                                            <label className="block text-xs font-bold text-slate-500 uppercase">
                                                <span className="mb-1.5 block">Tác giả (Author / Creator)</span>
                                                <input 
                                                    value={metadata.author} 
                                                    onChange={(e) => setMetadata((m) => ({ ...m, author: e.target.value }))} 
                                                    placeholder="Ví dụ: Tấm ốp than tre"
                                                    className="w-full rounded-xl border border-slate-200 px-3.5 py-2.5 text-xs outline-none transition focus:border-emerald-500 focus:ring-2 focus:ring-emerald-500/20" 
                                                />
                                            </label>

                                            <label className="block text-xs font-bold text-slate-500 uppercase">
                                                <span className="mb-1.5 block">Bản quyền (Copyright)</span>
                                                <input 
                                                    value={metadata.copyright} 
                                                    onChange={(e) => setMetadata((m) => ({ ...m, copyright: e.target.value }))} 
                                                    placeholder="Ví dụ: © 2026 TamOpThanTre"
                                                    className="w-full rounded-xl border border-slate-200 px-3.5 py-2.5 text-xs outline-none transition focus:border-emerald-500 focus:ring-2 focus:ring-emerald-500/20" 
                                                />
                                            </label>

                                            <label className="block text-xs font-bold text-slate-500 uppercase sm:col-span-2">
                                                <span className="mb-1 block">Từ khóa / Thẻ tag (Keywords)</span>
                                                <span className="mb-1.5 block text-[10px] text-slate-400 font-medium">Viết phân cách nhau bằng dấu phẩy</span>
                                                <input 
                                                    value={metadata.keywords} 
                                                    onChange={(e) => setMetadata((m) => ({ ...m, keywords: e.target.value }))} 
                                                    placeholder="tấm ốp than tre, thi công tấm ốp, tấm ốp vân sóng giá rẻ"
                                                    className="w-full rounded-xl border border-slate-200 px-3.5 py-2.5 text-xs outline-none transition focus:border-emerald-500 focus:ring-2 focus:ring-emerald-500/20" 
                                                />
                                            </label>

                                            <label className="block text-xs font-bold text-slate-500 uppercase">
                                                <span className="mb-1.5 block">Ngày thực hiện (Date Created)</span>
                                                <input 
                                                    type="date" 
                                                    value={metadata.date} 
                                                    onChange={(e) => setMetadata((m) => ({ ...m, date: e.target.value }))} 
                                                    className="w-full rounded-xl border border-slate-200 px-3.5 py-2 text-xs outline-none transition focus:border-emerald-500 focus:ring-2 focus:ring-emerald-500/20" 
                                                />
                                            </label>

                                            <div className="block text-xs font-bold text-slate-500 uppercase">
                                                <span className="mb-2 block">Xếp hạng chất lượng (Rating)</span>
                                                <div className="flex gap-1.5 py-1">
                                                    {[1, 2, 3, 4, 5].map((n) => (
                                                        <button 
                                                            key={n} 
                                                            onClick={() => setMetadata((m) => ({ ...m, rating: m.rating === n ? 0 : n }))} 
                                                            className="transform transition-transform active:scale-95" 
                                                            aria-label={`${n} sao`}
                                                        >
                                                            <svg className={`w-6 h-6 ${n <= metadata.rating ? "fill-amber-400 text-amber-400" : "text-slate-250 hover:text-amber-250"}`} viewBox="0 0 24 24" fill="currentColor">
                                                                <path d="M12 17.27L18.18 21l-1.64-7.03L22 9.24l-7.19-.61L12 2 9.19 8.63 2 9.24l5.46 4.73L5.82 21z"/>
                                                            </svg>
                                                        </button>
                                                    ))}
                                                </div>
                                            </div>
                                        </div>

                                        <label className="block text-xs font-bold text-slate-500 uppercase mt-2">
                                            <span className="mb-1.5 block">Mô tả tóm tắt (Comment)</span>
                                            <textarea 
                                                value={metadata.comment} 
                                                onChange={(e) => setMetadata((m) => ({ ...m, comment: e.target.value }))} 
                                                rows={3} 
                                                placeholder="Mô tả ngách sản phẩm dịch vụ..."
                                                className="w-full rounded-xl border border-slate-200 px-3.5 py-2.5 text-xs outline-none transition focus:border-emerald-500 focus:ring-2 focus:ring-emerald-500/20 resize-none" 
                                            />
                                        </label>
                                    </div>
                                )}

                                {/* Khối xử lý chính & Tải tệp xuống */}
                                <div className="rounded-2xl border border-slate-200 bg-white p-6 shadow-sm flex flex-col sm:flex-row items-center justify-between gap-4">
                                    <div className="text-center sm:text-left">
                                        <p className="text-xs text-slate-400 font-bold uppercase tracking-wider">Trạng thái xử lý ảnh</p>
                                        <p className="text-sm font-bold text-slate-700 mt-1">
                                            {files.length === 0 
                                                ? "Chưa nạp ảnh cần Geotag" 
                                                : `Sẵn sàng đóng gói ${files.length} tệp tin dạng ${outputFormat.toUpperCase()}`}
                                        </p>
                                    </div>

                                    {/* Nút bấm tải ảnh chính đã được cải tiến UX rõ ràng */}
                                    <button 
                                        disabled={!canSubmit} 
                                        onClick={handleSubmit} 
                                        className={`w-full sm:w-auto inline-flex items-center justify-center gap-2 rounded-xl px-8 py-4 font-bold text-white shadow-lg transition-all transform hover:-translate-y-0.5 active:translate-y-0 disabled:cursor-not-allowed disabled:opacity-60 disabled:transform-none ${
                                            canSubmit 
                                                ? "bg-slate-950 hover:bg-slate-900 shadow-slate-900/15" 
                                                : "bg-slate-400"
                                        }`}
                                    >
                                        {processing ? (
                                            <span className="h-5 w-5 animate-spin rounded-full border-3 border-white border-t-transparent" />
                                        ) : files.length > 1 ? (
                                            <svg className="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth="2"><path strokeLinecap="round" strokeLinejoin="round" d="M8 4H6a2 2 0 00-2 2v12a2 2 0 002 2h12a2 2 0 002-2V6a2 2 0 00-2-2h-2m-4-1v8m0 0l3-3m-3 3L9 8m-5 5h2.586a1 1 0 01.707.293l2.414 2.414a1 1 0 00.707.293h3.172a1 1 0 00.707-.293l2.414-2.414a1 1 0 01.707-.293H20" /></svg>
                                        ) : (
                                            <svg className="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth="2"><path strokeLinecap="round" strokeLinejoin="round" d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-4l-4 4m0 0l-4-4m4 4V4" /></svg>
                                        )}
                                        <span>
                                            {submitButtonText}
                                        </span>
                                    </button>
                                </div>

                            </section>
                        </div>
                    </main>

                    {/* Footer */}
                    <footer className="mt-20 border-t bg-white py-8 text-center text-xs text-slate-400 font-medium">
                        <p>© 2026 Geotag Image Optimization Tool. Bảo mật tuyệt đối, hoạt động trực tiếp trên trình duyệt Client-side.</p>
                    </footer>
                </div>
            );
        }

        // Khởi chạy ứng dụng React
        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<GeotagImageTool />);
    </script>
</body>
</html>
