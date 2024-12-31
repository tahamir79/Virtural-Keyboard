\documentclass[11pt]{article}

\usepackage[margin=1in]{geometry}
\usepackage{graphicx}
\usepackage{amsmath}
\usepackage{xcolor}
\usepackage{hyperref}

% TikZ (for flowchart)
\usepackage{tikz}
\usetikzlibrary{arrows.meta, shapes.geometric, positioning}

% === Code Listings ===
\usepackage{listings}
\lstset{
  language=JavaScript,       % detect JavaScript syntax
  frame=single,              % draw a frame around the code
  numbers=left,              % line numbers on the left
  numberstyle=\tiny\color{gray},
  numbersep=10pt,            % distance from code to line numbers
  xleftmargin=2em,           % indentation from the left margin
  keywordstyle=\color{blue}\bfseries,
  commentstyle=\color{green!60!black}\itshape,
  stringstyle=\color{red!70!black},
  basicstyle=\small\ttfamily,  % font settings for the code
  showstringspaces=false,      % do not visualize spaces in strings
  breaklines=true,             % allow breaking long lines
  breakatwhitespace=true,      % prefer to break at a space
  tabsize=2                    % default tabsize
}

\title{Eye-Tracking Keyboard Using a Webcam:\\
Comprehensive Deployment Introduction in Next.js}
\author{Taha Mir}
\date{\today}

\begin{document}
\maketitle

\begin{abstract}
This paper introduces a simple yet effective eye-tracking text-entry system powered by a standard webcam and MediaPipe FaceMesh. By estimating the user's gaze on an on-screen keyboard and detecting blinks via the Eye Aspect Ratio (EAR), the interface enables hands-free typing. Key parameters like blink thresholds and cooldown times can be tuned to accommodate different user preferences and environments. While straightforward to set up, this system is also extensible with more advanced techniques (e.g., GPU acceleration, sophisticated filtering) for those with deeper technical backgrounds.
\end{abstract}

\section{Introduction}
Eye-tracking can be a powerful modality for human-computer interaction, of particular interest to computer scientists and developers exploring novel interfaces. Although many existing eye-tracking solutions rely on specialized hardware, our system demonstrates that a standard webcam can power a fully functional, real-time eye-tracking keyboard using \textbf{MediaPipe FaceMesh} and a few calibration heuristics.

The core concepts include:
\begin{itemize}
  \item Detecting and tracking facial landmarks in real time using \textbf{MediaPipe FaceMesh}.
  \item Converting landmark positions into a 2D on-screen cursor position (``gaze cursor'').
  \item Detecting blinks via the \textbf{Eye Aspect Ratio (EAR)} for key presses.
  \item Tuning user parameters (e.g., blink threshold, cooldown time) to enhance usability.
\end{itemize}

This paper outlines the workflow, key parameters, code snippets, and deployment considerations in Next.js. We also discuss potential avenues to scale or refine the system for advanced use cases.

\section{High-Level Flow}
Figure~\ref{fig:flow} illustrates a simplified workflow:
\begin{enumerate}
  \item \textbf{Camera Feed:} The webcam captures frames.
  \item \textbf{Face Detection \& Landmarks:} MediaPipe FaceMesh identifies the eyes and other key points on the face.
  \item \textbf{Gaze Estimation:} We convert landmarks into a 2D on-screen cursor position, applying optional smoothing or filtering.
  \item \textbf{Blink Detection:} The Eye Aspect Ratio (EAR) checks if the user blinks.
  \item \textbf{Key Press:} If a blink is detected, the hovered key is ``pressed,'' updating the on-screen text.
\end{enumerate}

\begin{figure}[ht]
\centering
% ------------------ Flow chart starts here ------------------ %
\begin{tikzpicture}[
  font=\small,
  node distance=2.0cm,
  >=latex,
  every node/.style={align=center, font=\small},
  block/.style={
    rectangle, 
    draw,
    rounded corners,
    text width=5em,
    text centered,
    minimum height=2em
  },
  io/.style={
    trapezium, 
    draw,
    shape border rotate=180,
    trapezium left angle=70,
    trapezium right angle=110,
    text centered,
    minimum height=2em,
    text width=5em
  },
  line/.style={draw, -stealth}
]

% Nodes
\node[io] (camera) {Camera Frames};
\node[block, below=of camera] (facemesh) {MediaPipe \\ FaceMesh};
\node[block, below=of facemesh] (gaze) {Compute Gaze \\ (Cursor Position)};
\node[block, below=of gaze] (ear) {Check EAR \\ (Blink Detection)};
\node[block, below=of ear] (keyPress) {Key Press? \\ (If Blink)};
\node[block, below=of keyPress, text width=8em] (update) {Update Typed Text \\ \& UI Feedback};

% Lines
\draw[line] (camera) -- (facemesh);
\draw[line] (facemesh) -- (gaze);
\draw[line] (gaze) -- (ear);
\draw[line] (ear) -- (keyPress);
\draw[line] (keyPress) -- (update);

\end{tikzpicture}
% ------------------ Flow chart ends here ------------------ %
\caption{Core pipeline: Camera $\rightarrow$ FaceMesh $\rightarrow$ Gaze \& blink detection $\rightarrow$ Keyboard input.}
\label{fig:flow}
\end{figure}

\section{Implementation Details \& Performance Considerations}
For computer scientists, key architectural details may include:
\begin{itemize}
    \item \textbf{MediaPipe Integration:} MediaPipe FaceMesh runs primarily on the client side via WebAssembly or WebGL backends, depending on the browser. Where possible, enabling hardware acceleration can reduce CPU usage and improve frame rates.
    \item \textbf{Smoothing Filters:} By default, the raw gaze location will fluctuate due to slight head motions and landmark noise. Adding a low-pass filter or a Kalman filter (over \texttt{x,y} gaze coordinates) can significantly improve the user experience. 
    \item \textbf{EAR Computation Overhead:} EAR calculations are straightforward (a few vector subtractions and norms per frame) and typically do not bottleneck performance. However, if multiple face detections run concurrently (e.g., multi-user setups), you may need to parallelize or batch the computations.
    \item \textbf{Latency vs. Accuracy Trade-offs:} Lowering the input frame rate can reduce CPU/GPU usage but may delay blink detection. For real-time text entry, aim for at least 15--20\,FPS. 
    \item \textbf{Testing and Validation:} In a controlled environment, you can log the raw \texttt{(x, y)} positions and EAR values to analyze distribution patterns and refine thresholds.
\end{itemize}

\section{Key Parameters and Their Use}
\subsection{EAR Threshold}
The Eye Aspect Ratio threshold, \(\texttt{EAR\_THRESHOLD}\), determines when a blink is recognized. A typical default is \texttt{0.25}, but users can experiment with values between \texttt{0.20} and \texttt{0.30} depending on their blinking style and camera angle.

\subsection{Cooldown Time}
Once a blink triggers a key press, the system activates a cooldown (e.g., \texttt{1000 ms}). This prevents multiple key presses if the user holds their eyes closed or blinks rapidly. Adjusting this parameter may depend on observed user behavior.

\subsection{Gaze Scale}
A scaling factor (\texttt{GAZE\_SCALE}) maps small eye movements to broader on-screen cursor movements. Initial values near \texttt{5.0} are often sufficient, though advanced users may implement adaptive or user-specific calibration routines.

\section{Selected Code Snippets}

\subsection{States and Parameters in \texttt{page.js}}
Below is an abbreviated snippet from \texttt{page.js}, showcasing how we track typed text, cursor positions, and adjustable parameters in a typical React + Next.js environment:

\begin{lstlisting}
export default function Home() {
  // References to video/canvas
  const videoRef = useRef(null);
  const canvasRef = useRef(null);

  // Main states
  const [cursorPosition, setCursorPosition] = useState({x: 0, y: 0});
  const [typedText, setTypedText] = useState('');
  const [isShiftActive, setIsShiftActive] = useState(false);
  const [hoveredKey, setHoveredKey] = useState(null);

  // Prevent repeated triggers
  const cooldownRef = useRef(false);

  // Adjustable parameters
  const EAR_THRESHOLD = 0.25;   // Blink detection
  const GAZE_SCALE = 5.0;       // Cursor sensitivity
  const BLINK_COOLDOWN = 1000;  // in ms

  // ... other code for FaceMesh, event handling, etc. ...
}
\end{lstlisting}

\paragraph{Explanation:}
\begin{itemize}
  \item \texttt{EAR\_THRESHOLD}: Defines the cutoff for a blink event based on EAR.
  \item \texttt{BLINK\_COOLDOWN}: The lockout period during which repeated blinks do not generate additional presses.
  \item \texttt{GAZE\_SCALE}: Adjusts how eye movement maps to cursor displacement.
\end{itemize}

\subsection{Eye Aspect Ratio and Blink Detection}
\label{sec:ear}
We use the \emph{Eye Aspect Ratio (EAR)} formula:
\[
  EAR = \frac{\|p2 - p6\| + \|p3 - p5\|}{2 \times \|p1 - p4\|}
\]
Here, \(p1, p2, \dots, p6\) are eyelid landmarks. When the EAR for either eye drops below \(\texttt{EAR\_THRESHOLD}\), a blink is registered. The simplified pseudocode:

\begin{lstlisting}
function checkBlink(earLeft, earRight) {
  if (earLeft < EAR_THRESHOLD || earRight < EAR_THRESHOLD) {
    // If below threshold, treat as a blink
    return true;
  } else {
    return false;
  }
}
\end{lstlisting}

\paragraph{Action on Blink:} Once a blink is registered, we look up which key is hovered (via \texttt{document.elementFromPoint(x, y)}) and update \texttt{typedText} accordingly.

\subsection{Cooldown Mechanism}
To avoid repeated presses for a single extended blink, we enforce a cooldown:

\begin{lstlisting}
if (!cooldownRef.current) {
  // Perform keypress actions here...

  cooldownRef.current = true;
  setTimeout(() => {
    cooldownRef.current = false;
  }, BLINK_COOLDOWN);  // e.g., 1000 ms
}
\end{lstlisting}

\paragraph{User Customization:}
Changing the \texttt{BLINK\_COOLDOWN} value alters typing speed and reduces unintentional presses.

\section{Interface Components}
\subsection{\texttt{TextBox.jsx}}
Displays typed text and a blinking cursor:
\begin{itemize}
  \item \texttt{typedText}: The text entered so far.
  \item \texttt{isShiftActive}: React state controlling uppercase input.
  \item \texttt{Blinking Cursor}: A CSS-based animation, e.g., \texttt{animation: blink 1s step-start infinite}.
\end{itemize}

\subsection{\texttt{Keyboard.jsx}}
Renders an on-screen keyboard as a grid of \texttt{div} elements. Each element has a \texttt{data-value} attribute, allowing \texttt{document.elementFromPoint(x, y)} to identify the key under the gaze cursor.

\section{Deployment in Next.js}
\begin{itemize}
    \item \textbf{Client-Side Rendering (CSR):} Because this application relies on real-time webcam input and face-tracking, most logic must run in the browser. Consider disabling server-side rendering for the page(s) hosting the camera feed to avoid SSR conflicts.
    \item \textbf{Hosting \& Build Pipeline:} Standard Next.js deployment on platforms like Vercel or AWS Amplify works seamlessly. Ensure you configure the build so that environment variables (if any) or custom polyfills (e.g., WebAssembly hints) are properly set.
    \item \textbf{Security Considerations:} Browsers typically require HTTPS to allow camera access (getUserMedia). Thus, you must serve the app over a secure connection. 
    \item \textbf{Performance Logging:} If you need deeper insights, integrate \texttt{web-vitals} or a custom logging mechanism to measure latency, frame rate, and event timings.
\end{itemize}

\section{User Adjustments and Tips}
\begin{enumerate}
  \item \textbf{Adjust \texttt{EAR\_THRESHOLD}:} Typical range is \(\,0.20{-}0.30\). Empirical testing helps identify the threshold that aligns best with a particular user or camera angle.
  \item \textbf{Modify \texttt{BLINK\_COOLDOWN}:} Very short cooldowns may register multiple key presses during one prolonged blink. Longer cooldowns might slow typing speed.
  \item \textbf{Check Lighting Conditions}: Adequate, uniform lighting helps MediaPipe FaceMesh track eyes reliably. Backlit or low-contrast environments can degrade recognition accuracy.
  \item \textbf{Gaze Calibration}: If the cursor overshoots or undershoots frequently, tweak \texttt{GAZE\_SCALE} or implement a short calibration step to map gaze extremes more precisely.
\end{enumerate}

\section{Conclusion and Future Directions}
We presented a flexible, webcam-based eye-tracking keyboard that leverages MediaPipe FaceMesh for robust landmark detection, a straightforward EAR-based blink detection mechanism, and React states to manage dynamic UI updates. This approach highlights how accessible computer-vision-based assistive technologies can be developed without expensive hardware. 

Future enhancements may include:
\begin{itemize}
  \item \textbf{Adaptive Calibration}: Dynamically adjusting the mapping between facial landmarks and cursor position via machine learning models.
  \item \textbf{Improved Lighting Robustness}: Incorporating brightness checks or background modeling to improve landmark stability across varying illumination.
  \item \textbf{Advanced Layouts \& NLP}: Integrating multi-language keyboards, auto-complete, predictive text, and punctuation insertion.
  \item \textbf{Hybrid Multi-Modal Control}: Combining speech recognition or small head-movements with eye-tracking for faster text input and improved usability.
\end{itemize}

By offering an open framework and standardizing around a small set of adjustable parameters, this project facilitates rapid experimentation and a clear path toward more advanced solutions in gaze-based interfaces.

\end{document}
