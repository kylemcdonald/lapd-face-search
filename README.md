# LAPD Face Search

A web application that allows users to search through over 9,000 LAPD officer headshots using facial recognition technology. The application runs entirely in the browser, ensuring privacy by processing images locally without uploading them to any server.

## Features

- **Privacy-First**: All face recognition processing happens locally in your browser
- **Large Database**: Search through 9,000+ LAPD officer headshots
- **Real-Time Matching**: Upload a photo and get instant matches
- **Officer Information**: View officer names, serial numbers, and links to detailed profiles
- **Responsive Design**: Works on desktop and mobile devices

## How It Works

1. **Upload a Photo**: Select an image containing a face you want to identify
2. **Face Detection**: The app automatically detects faces in your uploaded image
3. **Feature Extraction**: Facial features are extracted and converted to mathematical descriptors
4. **Database Search**: Your face is compared against all officers in the database
5. **Results Display**: Top matches are shown with similarity scores and officer information

## Technology Stack

- **Frontend**: Svelte 5 with Vite
- **Face Recognition**: [@vladmandic/face-api](https://github.com/vladmandic/face-api) (TensorFlow.js-based)
- **Models**: SSD MobileNet v1 face detection, face landmark detection model, face recognition network
- **Data**: Pre-processed face descriptors and metadata for 9,000+ officers

## Project Structure

```
lapd-face-search/
├── public/
│   ├── analysis/
│   │   ├── lapd_descriptors.bin    # Pre-computed face descriptors
│   │   └── lapd_metadata.json      # Officer metadata (names, serials, etc.)
│   └── models/                     # TensorFlow.js face recognition models
├── src/
│   ├── App.svelte                  # Main application component
│   ├── Cop.svelte                  # Individual officer display component
│   ├── app.css                     # Global styles
│   └── main.js                     # Application entry point
├── index.html                      # HTML template
└── package.json                    # Dependencies and scripts
```

## Installation & Setup

1. **Clone the repository**:
   ```bash
   git clone <repository-url>
   cd lapd-face-search
   ```

2. **Install dependencies**:
   ```bash
   npm install
   ```

3. **Start development server**:
   ```bash
   npm run dev
   ```

4. **Build for production**:
   ```bash
   npm run build
   ```

## Usage

1. Open the application in your web browser
2. Wait for the face recognition models to load (this may take a few moments)
3. Click "Select Photo" to upload an image containing a face
4. The app will automatically process the image and display potential matches
5. Click on any match to view the officer's profile on WatchTheWatchers.net

## Privacy & Security

- **No Server Uploads**: Images are processed entirely in your browser
- **Local Processing**: Face recognition models run locally using TensorFlow.js
- **No Data Collection**: The application does not collect or store any uploaded images
- **Open Source**: All code is open source and can be audited for privacy compliance

## Data Sources

The officer database includes:
- Officer headshots from [public records](https://watchthewatchers.net/)
- Names and serial numbers
- Pre-computed facial descriptors for fast matching

## Technical Details

### Face Recognition Pipeline

1. **Face Detection**: Uses SSD MobileNet v1 to detect faces in images
2. **Landmark Detection**: Extracts 68 facial landmarks for alignment
3. **Feature Extraction**: Generates 128-dimensional face descriptors
4. **Similarity Matching**: Compares descriptors using Euclidean distance
5. **Ranking**: Returns top 9 matches sorted by similarity

### Performance

- **Initial Load**: ~50MB of models and data (one-time download)
- **Processing Speed**: ~1-3 seconds per image
- **Accuracy**: Optimized for clear, front-facing photos
- **Browser Support**: Modern browsers with WebGL support
