# Server.js-node-EC2-backend-API-
knfigurasi backend ec2








import express from "express";
import cors from "cors";
import crypto from "crypto";
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import { RekognitionClient, DetectLabelsCommand } from "@aws-sdk/client-rekognition";

const app = express();
app.use(express.json());

// mode public demo dulu
app.use(cors({ origin: "*" }));

const REGION = process.env.AWS_REGION || "us-east-1";
const UPLOAD_BUCKET = process.env.UPLOAD_BUCKET;
if (!UPLOAD_BUCKET) throw new Error("UPLOAD_BUCKET env is required");

const s3 = new S3Client({ region: REGION });
const rekog = new RekognitionClient({ region: REGION });

function makeKey(ct) {
  const ext = ct === "image/png" ? "png" : ct === "image/webp" ? "webp" : "jpg";
  return `uploads/${crypto.randomUUID()}.${ext}`;
}

app.get("/api/health", (req, res) => res.json({ ok: true }));

app.post("/api/uploads/presign", async (req, res) => {
  try {
    const { contentType } = req.body || {};
    const ct = String(contentType || "");

    const allowed = new Set(["image/jpeg", "image/png", "image/webp"]);
    if (!allowed.has(ct)) return res.status(400).send("Unsupported contentType");

    const key = makeKey(ct);
    const cmd = new PutObjectCommand({
      Bucket: UPLOAD_BUCKET,
      Key: key,
      ContentType: ct
    });

    const uploadUrl = await getSignedUrl(s3, cmd, { expiresIn: 120 });
    res.json({ uploadUrl, key });
  } catch (e) {
    res.status(500).send(String(e?.message || e));
  }
});

app.post("/api/rekognition/analyze", async (req, res) => {
  try {
    const { key } = req.body || {};
    if (!key || typeof key !== "string") return res.status(400).send("key required");
    if (!key.startsWith("uploads/")) return res.status(400).send("invalid key prefix");

    const cmd = new DetectLabelsCommand({
      Image: { S3Object: { Bucket: UPLOAD_BUCKET, Name: key } },
      MaxLabels: 10,
      MinConfidence: 70
    });

    const out = await rekog.send(cmd);
    res.json(out);
  } catch (e) {
    res.status(500).send(String(e?.message || e));
  }
});

app.listen(3000, () => console.log("API running on port 3000"));
