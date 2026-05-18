import React, { useMemo, useRef, useState, useEffect } from "react";
import { 
  Upload, MapPin, Download, Image as ImageIcon, Star, X, 
  FileArchive, LocateFixed, Search, CheckCircle, AlertTriangle, 
  Settings, Info, Map as MapIcon, Sparkles, Globe, Trash2
} from "lucide-react";

// Định nghĩa giới hạn tệp tin và định dạng được chấp nhận
const ACCEPTED_TYPES = ["image/jpeg", "image/png", "image/webp", "image/gif", "image/avif"];
const MAX_SIZE = 50 * 1024 * 1024; // 50MB

function clampNumber(value, min, max) {
  const n = Number(value);
  return Number.isFinite(n) && n >= min && n <= max;
}

function sanitizeFilename(name) {
  return name.replace(/\.[^.]+$/, "").replace(/[^a-z0-9-_]+/gi, "-").replace(/^-+|-+$/g, "") || "geotagged-image";
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

function encodeAscii(str = "") {
  const clean = String(str).replace(/[^\x00-\x7F]/g, "?");
  const bytes = new Uint8Array(clean.length + 1);
  for (let i = 0; i < clean.length; i++) bytes[i] = clean.charCodeAt(i);
  bytes[clean.length] = 0;
  return bytes;
}

function encodeUtf8(str = "") {
  return new TextEncoder().encode(String(str));
}

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

// Xây dựng phân đoạn EXIF dạng thô (TIFF) và phân đoạn JPEG APP1
function buildExif({ lat, lng, title, subject, keywords, copyright, author, date, comment, rating }) {
  const tiffHeaderLength = 8;
  const ifd0Offset = 8;
  const gpsOffsetPlaceholder = 0;
  const exifOffsetPlaceholder = 0;

  const ifd0Entries = [];
  const gpsEntries = [];
  const exifEntries = [];
  const dataBlocks = [];

  function addData(bytes) {
    dataBlocks.push(bytes);
    return 0; // Sẽ được cập nhật lại offset sau
  }

  function entry(tag, type, count, valueOrOffset) {
    return { tag, type, count, valueOrOffset };
  }

  if (title) ifd0Entries.push(entry(0x010e, 2, encodeAscii(title).length, addData(encodeAscii(title)))); // ImageDescription
  if (author) ifd0Entries.push(entry(0x013b, 2, encodeAscii(author).length, addData(encodeAscii(author)))); // Artist
  if (copyright) ifd0Entries.push(entry(0x8298, 2, encodeAscii(copyright).length, addData(encodeAscii(copyright))));
  if (rating) ifd0Entries.push(entry(0x4746, 3, 1, Number(rating))); // Rating cho Windows / Lightroom

  ifd0Entries.push(entry(0x8825, 4, 1, gpsOffsetPlaceholder)); // Con trỏ tới GPSInfoIFD
  ifd0Entries.push(entry(0x8769, 4, 1, exifOffsetPlaceholder)); // Con trỏ tới ExifIFD

  const latDms = decimalToDmsRationals(lat);
  const lngDms = decimalToDmsRationals(lng);
  gpsEntries.push(entry(0x0000, 1, 4, 0x02030000)); // Phiên bản GPS 2.3.0.0
  gpsEntries.push(entry(0x0001, 2, 2, addData(encodeAscii(Number(lat) >= 0 ? "N" : "S"))));
  gpsEntries.push(entry(0x0002, 5, 3, addData(rationalsToBytes(latDms))));
  gpsEntries.push(entry(0x0003, 2, 2, addData(encodeAscii(Number(lng) >= 0 ? "E" : "W"))));
  gpsEntries.push(entry(0x0004, 5, 3, addData(rationalsToBytes(lngDms))));
  if (date) gpsEntries.push(entry(0x001d, 2, encodeAscii(date).length, addData(encodeAscii(date)))); // GPSDateStamp

  const userComment = [title, subject, keywords, comment].filter(Boolean).join(" | ");
  if (userComment) exifEntries.push(entry(0x9286, 7, encodeUtf8("ASCII\0\0\0" + userComment).length, addData(encodeUtf8("ASCII\0\0\0" + userComment))));
  if (date) {
    const exifDate = String(date).replace(/-/g, ":") + " 00:00:00";
    exifEntries.push(entry(0x9003, 2, encodeAscii(exifDate).length, addData(encodeAscii(exifDate))));
  }

  const ifd0Size = 2 + ifd0Entries.length * 12 + 4;
  const gpsOffset = ifd0Offset + ifd0Size;
  const gpsSize = 2 + gpsEntries.length * 12 + 4;
  const exifOffset = gpsOffset + gpsSize;
  const exifSize = 2 + exifEntries.length * 12 + 4;
  let dataOffset = exifOffset + exifSize;

  const patchedDataOffsets = dataBlocks.map((block) => {
    const offset = dataOffset;
    dataOffset += block.length;
    return offset;
  });
  let dataIndex = 0;

  function serializeIFD(entries, pointerPatch = {}) {
    const bytes = new Uint8Array(2 + entries.length * 12 + 4);
    const view = new DataView(bytes.buffer);
    view.setUint16(0, entries.length, true);
    entries.forEach((e, idx) => {
      const base = 2 + idx * 12;
      view.setUint16(base, e.tag, true);
      view.setUint16(base + 2, e.type, true);
      view.setUint32(base + 4, e.count, true);
      let value = e.valueOrOffset;
      if (e.tag === 0x8825) value = pointerPatch.gpsOffset;
      else if (e.tag === 0x8769) value = pointerPatch.exifOffset;
      else if (value === 0) value = patchedDataOffsets[dataIndex++];

      if (e.type === 3 && e.count === 1) view.setUint16(base + 8, value, true);
      else if (e.type === 1 && e.count <= 4) view.setUint32(base + 8, value, false);
      else view.setUint32(base + 8, value, true);
    });
    view.setUint32(bytes.length - 4, 0, true);
    return bytes;
  }

  function rationalsToBytes(rationals) {
    const bytes = new Uint8Array(rationals.length * 8);
    const view = new DataView(bytes.buffer);
    rationals.forEach(([num, den], i) => {
      view.setUint32(i * 8, num, true);
      view.setUint32(i * 8 + 4, den, true);
    });
    return bytes;
  }

  const tiffHeader = new Uint8Array(tiffHeaderLength);
  const tiffView = new DataView(tiffHeader.buffer);
  tiffHeader[0] = 0x49; // Byte order: Little-endian (I)
  tiffHeader[1] = 0x49;
  tiffView.setUint16(2, 42, true);
  tiffView.setUint32(4, ifd0Offset, true);

  dataIndex = 0;
  const ifd0 = serializeIFD(ifd0Entries, { gpsOffset, exifOffset });
  const gpsIFD = serializeIFD(gpsEntries);
  const exifIFD = serializeIFD(exifEntries);
  const tiff = concatBytes(tiffHeader, ifd0, gpsIFD, exifIFD, ...dataBlocks);
  
  // Bản ghi đóng gói cho JPEG APP1 segment
  const exifHeader = encodeUtf8("Exif\0\0");
  const payload = concatBytes(exifHeader, tiff);
  const length = payload.length + 2;
  const jpegSegment = concatBytes(new Uint8Array([0xff, 0xe1]), writeU16BE(length), payload);

  return { jpegSegment, tiffBytes: tiff };
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

// Phân tích nhị phân và xử lý siêu dữ liệu cho định dạng WebP (RIFF WebP Container)
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
      // Lossy format
      if (width === 0) {
        if (chunkData[3] === 0x9d && chunkData[4] === 0x01 && chunkData[5] === 0x2a) {
          width = (chunkData[6] | (chunkData[7] << 8)) & 0x3fff;
          height = (chunkData[8] | (chunkData[9] << 8)) & 0x3fff;
        }
      }
      imageChunks.push({ tag, data: chunkData });
    } else if (tag === "VP8L") {
      // Lossless format
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
      // Bỏ qua các khối EXIF/XMP cũ để tránh xung đột dữ liệu
    } else {
      otherChunks.push({ tag, data: chunkData });
    }

    offset += 8 + size;
    if (size % 2 !== 0) offset++; // Padding byte RIFF
  }

  return { width, height, hasAlpha, hasICC, imageChunks, otherChunks };
}

function writeWebPChunk(tag, data) {
  const tagBytes = encodeAscii(tag).slice(0, 4);
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

// Chèn Exif TIFF vào cấu trúc nhị phân của WebP
function insertExifIntoWebp(webpBytes, tiffBytes, fallbackWidth, fallbackHeight) {
  const parsed = parseWebP(webpBytes);
  const width = parsed.width || fallbackWidth || 800;
  const height = parsed.height || fallbackHeight || 600;

  // Xây dựng payload VP8X theo chuẩn Extended WebP
  const vp8xPayload = new Uint8Array(10);
  let flags = 0x08; // Kích hoạt cờ EXIF (0x08)
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
  // Giữ lại profile ICC nếu có
  parsed.otherChunks.forEach(c => {
    if (c.tag === "ICCP") middleChunks.push(writeWebPChunk(c.tag, c.data));
  });
  // Thêm dữ liệu ảnh chính
  parsed.imageChunks.forEach(c => {
    middleChunks.push(writeWebPChunk(c.tag, c.data));
  });
  // Thêm các chunk không liên quan khác
  parsed.otherChunks.forEach(c => {
    if (c.tag !== "ICCP") middleChunks.push(writeWebPChunk(c.tag, c.data));
  });

  // Tạo chunk EXIF mới
  const exifChunk = writeWebPChunk("EXIF", tiffBytes);

  // Đóng gói hoàn chỉnh WebP
  const payload = concatBytes(vp8xChunk, ...middleChunks, exifChunk);

  const riffHeader = new Uint8Array(12);
  riffHeader.set(encodeAscii("RIFF").slice(0, 4), 0);
  const totalSize = payload.length + 4; // thêm 4 byte cho ký hiệu "WEBP"
  riffHeader[4] = totalSize & 255;
  riffHeader[5] = (totalSize >> 8) & 255;
  riffHeader[6] = (totalSize >> 16) & 255;
  riffHeader[7] = (totalSize >> 24) & 255;
  riffHeader.set(encodeAscii("WEBP").slice(0, 4), 8);

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

function makePngTextChunk(keyword, text) {
  const type = encodeUtf8("tEXt");
  const data = concatBytes(encodeUtf8(keyword), new Uint8Array([0]), encodeUtf8(String(text || "")));
  const crc = crc32(concatBytes(type, data));
  return concatBytes(writeU32BE(data.length), type, data, writeU32BE(crc));
}

function insertTextChunksIntoPng(pngBytes, metadata) {
  const signature = pngBytes.slice(0, 8);
  const chunks = Object.entries(metadata)
    .filter(([, value]) => value !== undefined && value !== null && String(value).trim() !== "")
    .map(([key, value]) => makePngTextChunk(key, value));
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

function parseCoordinates(input) {
  const text = String(input || "").trim();
  const at = text.match(/@(-?\d+\.\d+),(-?\d+\.\d+)/);
  if (at) return { lat: at[1], lng: at[2] };
  const pair = text.match(/(-?\d+(?:\.\d+)?)\s*,\s*(-?\d+(?:\.\d+)?)/);
  if (pair) return { lat: pair[1], lng: pair[2] };
  return null;
}

export default function GeotagImageTool() {
  const inputRef = useRef(null);
  const mapRef = useRef(null);
  const mapInstance = useRef(null);
  const markerInstance = useRef(null);
  const shouldPanMap = useRef(false);

  // States quản lý tệp hình ảnh & vị trí
  const [files, setFiles] = useState([]);
  const [dragging, setDragging] = useState(false);
  const [lat, setLat] = useState("21.028511"); // Hà Nội
  const [lng, setLng] = useState("105.854223");
  const [mapsText, setMapsText] = useState("");
  const [searchQuery, setSearchQuery] = useState("");
  const [searchingAddress, setSearchingAddress] = useState(false);
  
  // Trạng thái các thư viện liên kết
  const [mapReady, setMapReady] = useState(false);
  const [zipReady, setZipReady] = useState(false);

  // Trình quản lý Tab
  const [activeTab, setActiveTab] = useState("location"); // "location", "seo"

  // Cấu hình định dạng xuất bản
  const [outputFormat, setOutputFormat] = useState("jpg"); // "jpg", "png", "webp"
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

  // Tải Leaflet Map và JSZip không đồng bộ để đạt độ tin cậy tối đa
  useEffect(() => {
    if (!window.JSZip) {
      const zipScript = document.createElement("script");
      zipScript.src = "https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js";
      zipScript.async = true;
      zipScript.onload = () => setZipReady(true);
      document.body.appendChild(zipScript);
    } else {
      setZipReady(true);
    }

    if (!window.L) {
      const leafletCss = document.createElement("link");
      leafletCss.rel = "stylesheet";
      leafletCss.href = "https://unpkg.com/leaflet@1.9.4/dist/leaflet.css";
      document.head.appendChild(leafletCss);

      const leafletJs = document.createElement("script");
      leafletJs.src = "https://unpkg.com/leaflet@1.9.4/dist/leaflet.js";
      leafletJs.async = true;
      leafletJs.onload = () => setMapReady(true);
      document.body.appendChild(leafletJs);
    } else {
      setMapReady(true);
    }
  }, []);

  // Đồng bộ hóa Leaflet map với các dịch chuyển tọa độ
  useEffect(() => {
    if (!mapReady || !mapRef.current || !window.L) return;

    if (!mapInstance.current) {
      mapInstance.current = window.L.map(mapRef.current, {
        zoomControl: true,
        scrollWheelZoom: true
      }).setView([parseFloat(lat) || 21.0285, parseFloat(lng) || 105.8542], 13);

      window.L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '© OpenStreetMap contributors'
      }).addTo(mapInstance.current);

      mapInstance.current.on('click', (e) => {
        const { lat, lng } = e.latlng;
        setLat(lat.toFixed(7));
        setLng(lng.toFixed(7));
        showToast("info", `Đã cập nhật tọa độ mới từ bản đồ: ${lat.toFixed(5)}, ${lng.toFixed(5)}`);
      });
    }

    const currentLat = parseFloat(lat);
    const currentLng = parseFloat(lng);

    if (!isNaN(currentLat) && !isNaN(currentLng)) {
      const latlng = [currentLat, currentLng];

      if (!markerInstance.current) {
        markerInstance.current = window.L.marker(latlng, { 
          draggable: true 
        }).addTo(mapInstance.current);

        markerInstance.current.on('dragend', (e) => {
          const position = e.target.getLatLng();
          setLat(position.lat.toFixed(7));
          setLng(position.lng.toFixed(7));
          showToast("info", `Đã di chuyển ghim: ${position.lat.toFixed(5)}, ${position.lng.toFixed(5)}`);
        });
      } else {
        markerInstance.current.setLatLng(latlng);
      }

      if (shouldPanMap.current) {
        mapInstance.current.setView(latlng, 14);
        shouldPanMap.current = false;
      }
    }
  }, [mapReady, lat, lng]);

  const previews = useMemo(() => files.map((file) => ({ file, url: URL.createObjectURL(file) })), [files]);
  const canSubmit = files.length > 0 && clampNumber(lat, -90, 90) && clampNumber(lng, -180, 180) && !processing;

  function showToast(type, text) {
    setMessage({ type, text });
    setTimeout(() => {
      setMessage((current) => current.text === text ? { type: "", text: "" } : current);
    }, 6000);
  }

  // Phân giải từ khóa địa chỉ sang GPS sử dụng Nominatim OpenStreetMap API tự do
  async function searchAddress(query) {
    if (!query || !query.trim()) {
      showToast("error", "Vui lòng nhập từ khóa tìm kiếm địa điểm!");
      return;
    }
    setSearchingAddress(true);
    try {
      const res = await fetch(`https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(query)}&limit=1`);
      const data = await res.json();
      if (data && data.length > 0) {
        const first = data[0];
        shouldPanMap.current = true;
        setLat(parseFloat(first.lat).toFixed(7));
        setLng(parseFloat(first.lon).toFixed(7));
        showToast("success", `Địa điểm tìm thấy: ${first.display_name}`);
      } else {
        showToast("error", "Không tìm thấy địa điểm này. Vui lòng nhập từ khóa chi tiết hơn.");
      }
    } catch (e) {
      showToast("error", "Lỗi dịch vụ tìm kiếm. Vui lòng thử nhập thủ công tọa độ.");
    } finally {
      setSearchingAddress(false);
    }
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
        shouldPanMap.current = true;
        setLat(pos.coords.latitude.toFixed(7));
        setLng(pos.coords.longitude.toFixed(7));
        showToast("success", "Đã định vị thành công vị trí hiện tại.");
      },
      () => showToast("error", "Không thể lấy vị trí hiện tại. Hãy kiểm tra lại cấp quyền GPS.")
    );
  }

  function applyMapsText() {
    const parsed = parseCoordinates(mapsText);
    if (!parsed) {
      showToast("error", "Không trích xuất được thông tin tọa độ phù hợp.");
      return;
    }
    shouldPanMap.current = true;
    setLat(parsed.lat);
    setLng(parsed.lng);
    showToast("success", `Đã áp dụng tọa độ: ${parsed.lat}, ${parsed.lng}`);
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
        processed.push({ blob, filename: `${sanitizeFilename(file.name)}-geotag.${ext}` });
      }

      if (processed.length === 1) {
        downloadBlob(processed[0].blob, processed[0].filename);
        showToast("success", "Đã đóng gói dữ liệu và tự động tải về ảnh!");
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
      {/* Thông báo nổi */}
      {message.text && (
        <div className="fixed top-5 right-5 z-[9999] max-w-md animate-bounce-short rounded-xl border bg-white p-4 shadow-xl transition-all">
          <div className="flex items-start gap-3">
            {message.type === "success" && <CheckCircle className="text-emerald-500 shrink-0" size={20} />}
            {message.type === "error" && <AlertTriangle className="text-rose-500 shrink-0" size={20} />}
            {message.type === "info" && <Info className="text-indigo-500 shrink-0" size={20} />}
            <div>
              <p className="text-sm font-semibold text-slate-900">
                {message.type === "success" ? "Thành công" : message.type === "error" ? "Gặp sự cố" : "Thông báo"}
              </p>
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
                <Globe size={28} className="animate-spin-slow" />
              </div>
              <div>
                <h1 className="text-2xl sm:text-3xl font-extrabold text-slate-900 tracking-tight">Geotag & SEO Metadata Tool</h1>
                <p className="mt-1 text-sm text-slate-500">
                  Nhúng tọa độ định vị GPS, thông tin thương hiệu và Metadata nâng cao vào ảnh JPG, PNG hoặc WEBP để tối ưu SEO hình ảnh.
                </p>
              </div>
            </div>
            <div className="flex flex-wrap gap-2">
              <span className="inline-flex items-center gap-1.5 rounded-full bg-emerald-50 px-3 py-1 text-xs font-semibold text-emerald-700 border border-emerald-100">
                <span className="h-1.5 w-1.5 rounded-full bg-emerald-500 animate-ping" />
                Chạy 100% Client-side (Bảo mật tuyệt đối)
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
                  <ImageIcon size={20} className="text-emerald-600" />
                  <span>Hình Ảnh Đầu Vào ({files.length})</span>
                </div>
                {files.length > 0 && (
                  <button 
                    onClick={clearAllFiles}
                    className="flex items-center gap-1 text-xs font-medium text-rose-500 hover:text-rose-700 transition"
                  >
                    <Trash2 size={13} />
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
                <Upload className="mx-auto mb-3 text-emerald-500 animate-pulse" size={40} />
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

              {/* Hiển thị danh sách ảnh đã nạp */}
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
                        <X size={14} />
                      </button>
                    </div>
                  ))}
                </div>
              ) : (
                <div className="mt-6 flex flex-col items-center justify-center py-8 text-center border border-dashed rounded-xl bg-slate-50/40 text-slate-400">
                  <ImageIcon size={32} className="stroke-1 text-slate-300 mb-2" />
                  <p className="text-xs">Chưa có ảnh nào được chọn</p>
                  <p className="text-[10px] mt-0.5 max-w-[200px]">Hãy kéo thả hoặc tải lên ít nhất một tệp để bắt đầu gán tọa độ.</p>
                </div>
              )}
            </div>

            {/* Cấu hình Định dạng đầu ra */}
            <div className="rounded-2xl border border-slate-200 bg-white p-6 shadow-sm">
              <div className="mb-4 flex items-center gap-2 font-bold text-slate-800">
                <Settings size={20} className="text-indigo-500" />
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
                      <Info size={16} className="text-indigo-500 shrink-0" />
                      <span>
                        <strong>Được đề xuất nhiều nhất:</strong> Định dạng JPG sẽ được ghi đè metadata trực tiếp theo cấu trúc chuẩn Exif APP1, hỗ trợ đồng nhất hoàn hảo trên Google Maps, Windows File Explorer, Photoshop, Lightroom, v.v.
                      </span>
                    </div>
                  )}
                  {outputFormat === "png" && (
                    <div className="flex gap-2">
                      <Info size={16} className="text-indigo-500 shrink-0" />
                      <span>
                        <strong>PNG Chunk Metadata:</strong> Do cấu trúc lưu trữ PNG, hệ thống sẽ chèn dữ liệu dạng khối văn bản tEXt (tEXt Chunks). Một số hệ thống bên thứ ba có thể sẽ không đọc đầy đủ định dạng này như dạng EXIF nguyên bản.
                      </span>
                    </div>
                  )}
                  {outputFormat === "webp" && (
                    <div className="flex gap-2">
                      <Info size={16} className="text-teal-600 shrink-0" />
                      <span>
                        <strong>Chuẩn WebP thế hệ mới:</strong> Tiết kiệm tới hơn 30% dung lượng mà vẫn giữ nguyên độ trong suốt và sắc nét tuyệt hảo. Hệ thống sẽ nâng cấp tệp WebP lên định dạng Extended (VP8X) để nhúng trực tiếp khối dữ liệu nhị phân **EXIF GPS** cao cấp của JPG!
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
                { id: "location", label: "Bản Đồ & Tọa Độ Vị Trí", icon: MapIcon, color: "text-emerald-500" },
                { id: "seo", label: "Metadata SEO Doanh Nghiệp", icon: Sparkles, color: "text-amber-500" }
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
                    <Icon size={16} className={activeTab === tab.id ? "text-white" : tab.color} />
                    <span>{tab.label}</span>
                  </button>
                );
              })}
            </div>

            {/* TAB 1: BẢN ĐỒ & TỌA ĐỘ */}
            {activeTab === "location" && (
              <div className="rounded-2xl border border-slate-200 bg-white p-6 shadow-sm space-y-6 animate-fadeIn">
                <div className="flex flex-col sm:flex-row sm:items-center justify-between gap-3">
                  <div className="flex items-center gap-2 font-bold text-slate-800">
                    <MapPin size={20} className="text-emerald-500" />
                    <span>Xác Định Tọa Độ Địa Lý (GPS)</span>
                  </div>
                  <button 
                    onClick={useCurrentLocation}
                    className="inline-flex items-center justify-center gap-1.5 rounded-xl border border-slate-200 bg-white px-3 py-1.5 text-xs font-bold text-slate-700 shadow-sm transition hover:bg-slate-50"
                  >
                    <LocateFixed size={14} className="text-emerald-500" />
                    Lấy vị trí hiện tại
                  </button>
                </div>

                {/* Ô tìm kiếm thông minh */}
                <div className="space-y-2">
                  <label className="text-xs font-bold text-slate-500 block uppercase">Tìm kiếm địa chỉ nhanh</label>
                  <div className="relative">
                    <input 
                      type="text" 
                      value={searchQuery}
                      onChange={(e) => setSearchQuery(e.target.value)}
                      onKeyDown={(e) => e.key === 'Enter' && searchAddress(searchQuery)}
                      placeholder="Nhập địa điểm: 'Nhà thờ Đức Bà', 'Hồ Hoàn Kiếm'..." 
                      className="w-full rounded-xl border border-slate-200 pl-4 pr-24 py-3 text-sm outline-none transition focus:border-emerald-500 focus:ring-2 focus:ring-emerald-500/20"
                    />
                    <button 
                      onClick={() => searchAddress(searchQuery)}
                      disabled={searchingAddress}
                      className="absolute right-2 top-2 inline-flex items-center gap-1 rounded-lg bg-emerald-600 px-3 py-1.5 text-xs font-bold text-white hover:bg-emerald-700 disabled:opacity-50 transition"
                    >
                      {searchingAddress ? (
                        <span className="h-3 w-3 animate-spin rounded-full border-2 border-white border-t-transparent" />
                      ) : (
                        <Search size={13} />
                      )}
                      Tìm vị trí
                    </button>
                  </div>
                </div>

                {/* Bản đồ tương tác Leaflet */}
                <div className="space-y-2">
                  <div className="flex items-center justify-between">
                    <label className="text-xs font-bold text-slate-500 block uppercase">Bản đồ vệ tinh trực quan</label>
                    <span className="text-[10px] text-slate-400 font-medium">Click chọn điểm trên bản đồ hoặc kéo thả Ghim màu xanh</span>
                  </div>
                  <div className="relative overflow-hidden rounded-2xl border border-slate-200 shadow-inner">
                    <div ref={mapRef} className="h-72 w-full z-10" />
                    {!mapReady && (
                      <div className="absolute inset-0 z-20 flex flex-col items-center justify-center bg-slate-100 text-slate-400">
                        <span className="h-8 w-8 animate-spin rounded-full border-4 border-emerald-500 border-t-transparent" />
                        <p className="mt-2 text-xs">Đang nạp dữ liệu bản đồ...</p>
                      </div>
                    )}
                  </div>
                </div>

                {/* Trường hiển thị tọa độ */}
                <div className="grid gap-4 sm:grid-cols-2">
                  <label className="block text-xs font-bold text-slate-500 uppercase">
                    <span className="mb-2 block">Vĩ độ (Latitude) *</span>
                    <input 
                      value={lat} 
                      onChange={(e) => setLat(e.target.value)} 
                      type="number" 
                      step="any" 
                      placeholder="-90 đến 90" 
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
                      placeholder="-180 đến 180" 
                      className="w-full rounded-xl border border-slate-200 px-3.5 py-3 text-sm font-semibold text-slate-800 outline-none transition focus:border-emerald-500 focus:ring-2 focus:ring-emerald-500/20" 
                    />
                  </label>
                </div>

                {/* Trích xuất tọa độ cao cấp */}
                <div className="border-t pt-4 space-y-2">
                  <label className="text-xs font-bold text-slate-500 block uppercase">Trích xuất tọa độ tự động</label>
                  <div className="flex flex-col sm:flex-row gap-2">
                    <input 
                      value={mapsText} 
                      onChange={(e) => setMapsText(e.target.value)} 
                      placeholder="Dán link Google Maps hoặc chuỗi '10.7769, 106.7009'..." 
                      className="flex-1 rounded-xl border border-slate-200 px-3.5 py-2.5 text-xs outline-none transition focus:border-emerald-500 focus:ring-2 focus:ring-emerald-500/20" 
                    />
                    <button 
                      onClick={applyMapsText} 
                      className="rounded-xl border border-slate-200 bg-white px-4 py-2.5 text-xs font-bold text-slate-700 shadow-sm transition hover:bg-slate-50"
                    >
                      Quét tọa độ
                    </button>
                  </div>
                </div>
              </div>
            )}

            {/* TAB 2: SEO & METADATA DOANH NGHIỆP */}
            {activeTab === "seo" && (
              <div className="rounded-2xl border border-slate-200 bg-white p-6 shadow-sm space-y-4 animate-fadeIn">
                <div className="mb-2 flex items-center gap-2 font-bold text-slate-800 border-b pb-3">
                  <Sparkles size={20} className="text-amber-500 animate-pulse" />
                  <span>Cấu Hình Metadata SEO Doanh Nghiệp (Không bắt buộc)</span>
                </div>

                <div className="grid gap-4 sm:grid-cols-2">
                  <label className="block text-xs font-bold text-slate-500 uppercase">
                    <span className="mb-1.5 block">Tiêu đề ảnh (Title)</span>
                    <input 
                      value={metadata.title} 
                      onChange={(e) => setMetadata((m) => ({ ...m, title: e.target.value }))} 
                      placeholder="Ví dụ: Thiết Kế Phòng Ngủ Gỗ Sồi Cao Cấp"
                      className="w-full rounded-xl border border-slate-200 px-3.5 py-2.5 text-xs outline-none transition focus:border-emerald-500 focus:ring-2 focus:ring-emerald-500/20" 
                    />
                  </label>

                  <label className="block text-xs font-bold text-slate-500 uppercase">
                    <span className="mb-1.5 block">Chủ đề (Subject)</span>
                    <input 
                      value={metadata.subject} 
                      onChange={(e) => setMetadata((m) => ({ ...m, subject: e.target.value }))} 
                      placeholder="Ví dụ: Trang trí nội thất nhà phố"
                      className="w-full rounded-xl border border-slate-200 px-3.5 py-2.5 text-xs outline-none transition focus:border-emerald-500 focus:ring-2 focus:ring-emerald-500/20" 
                    />
                  </label>

                  <label className="block text-xs font-bold text-slate-500 uppercase">
                    <span className="mb-1.5 block">Tác giả (Author / Creator)</span>
                    <input 
                      value={metadata.author} 
                      onChange={(e) => setMetadata((m) => ({ ...m, author: e.target.value }))} 
                      placeholder="Ví dụ: Tên thương hiệu / Công ty dịch vụ của bạn"
                      className="w-full rounded-xl border border-slate-200 px-3.5 py-2.5 text-xs outline-none transition focus:border-emerald-500 focus:ring-2 focus:ring-emerald-500/20" 
                    />
                  </label>

                  <label className="block text-xs font-bold text-slate-500 uppercase">
                    <span className="mb-1.5 block">Bản quyền (Copyright)</span>
                    <input 
                      value={metadata.copyright} 
                      onChange={(e) => setMetadata((m) => ({ ...m, copyright: e.target.value }))} 
                      placeholder="Ví dụ: © 2026 ThuongHieuLtd"
                      className="w-full rounded-xl border border-slate-200 px-3.5 py-2.5 text-xs outline-none transition focus:border-emerald-500 focus:ring-2 focus:ring-emerald-500/20" 
                    />
                  </label>

                  <label className="block text-xs font-bold text-slate-500 uppercase sm:col-span-2">
                    <span className="mb-1 block">Từ khóa / Thẻ tag (Keywords)</span>
                    <span className="mb-1.5 block text-[10px] text-slate-400 font-medium">Viết phân cách nhau bằng dấu phẩy để hệ thống tự động nhận dạng</span>
                    <input 
                      value={metadata.keywords} 
                      onChange={(e) => setMetadata((m) => ({ ...m, keywords: e.target.value }))} 
                      placeholder="thiết kế nội thất, thi công chung cư hà nội, báo giá tủ gỗ sồi"
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
                          <Star 
                            size={24} 
                            className={n <= metadata.rating ? "fill-amber-400 text-amber-400 drop-shadow" : "text-slate-200 hover:text-amber-200"} 
                          />
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
                    placeholder="Mô tả ngách sản phẩm dịch vụ hoặc giới thiệu tóm tắt thông tin doanh nghiệp..."
                    className="w-full rounded-xl border border-slate-200 px-3.5 py-2.5 text-xs outline-none transition focus:border-emerald-500 focus:ring-2 focus:ring-emerald-500/20 resize-none" 
                  />
                </label>
              </div>
            )}

            {/* Khối xử lý chính */}
            <div className="rounded-2xl border border-slate-200 bg-white p-6 shadow-sm flex flex-col sm:flex-row items-center justify-between gap-4">
              <div className="text-center sm:text-left">
                <p className="text-xs text-slate-400 font-bold uppercase tracking-wider">Trạng thái xử lý ảnh</p>
                <p className="text-sm font-bold text-slate-700 mt-1">
                  {files.length === 0 
                    ? "Chưa nạp ảnh cần Geotag" 
                    : `Sẵn sàng đóng gói ${files.length} tệp tin dạng ${outputFormat.toUpperCase()}`}
                </p>
              </div>

              <button 
                disabled={!canSubmit} 
                onClick={handleSubmit} 
                className={`w-full sm:w-auto inline-flex items-center justify-center gap-2 rounded-xl bg-slate-950 px-8 py-4 font-bold text-white shadow-lg transition-all transform hover:-translate-y-0.5 active:translate-y-0 disabled:cursor-not-allowed disabled:opacity-50 disabled:transform-none ${
                  canSubmit ? "hover:bg-slate-900 shadow-slate-900/15" : ""
                }`}
              >
                {processing ? (
                  <span className="h-5 w-5 animate-spin rounded-full border-3 border-white border-t-transparent" />
                ) : files.length > 1 ? (
                  <FileArchive size={18} />
                ) : (
                  <Download size={18} />
                )}
                <span>
                  {processing ? "Đang xử lý cấu trúc ảnh..." : files.length > 1 ? "Ghi đè & Tải về gói ZIP" : "Ghi đè & Tải về ảnh"}
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
