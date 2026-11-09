# ==========================================
# STAGE 1: Builder
# ==========================================
FROM python:3.10-slim-trixie AS builder

ARG COMMIT=6e1036dbfcde1c293b50c742f0736a3965dd1e0d

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential \
        git \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /base

# Use Python's built-in venv instead of downloading the outdated virtualenv package
RUN python -m venv /venv
ENV PATH="/venv/bin:$PATH"

# Clone, install cleanly, and discard source
# Replaced `setup.py develop` with a standard pip install so we don't need the source files
RUN git clone https://github.com/deepmipt/DeepPavlov.git \
    && cd DeepPavlov \
    && git checkout $COMMIT \
    && pip install --no-cache-dir . \
    && cd /base && rm -rf DeepPavlov

RUN pip install --no-cache-dir email-validator lxml

# Copy configuration for installation and downloading models
COPY config/train_tfidf_logreg_en_faq.json /base/config/

# Install dependencies, pin scikit-learn==1.0.2 (+ compatible numpy/scipy), and download weights
RUN python -m deeppavlov install /base/config/train_tfidf_logreg_en_faq.json \
    && pip install --no-cache-dir "scikit-learn==1.0.2" "scipy<1.10.0" "numpy<1.24.0" \
    && python -m deeppavlov download /base/config/train_tfidf_logreg_en_faq.json

# Pre-download NLTK packages so offline runtime doesn't fail with Errno 101
ENV NLTK_DATA=/usr/local/share/nltk_data
RUN python -m nltk.downloader -d /usr/local/share/nltk_data \
    punkt \
    punkt_tab \
    stopwords \
    perluniprops \
    nonbreaking_prefixes

# ==========================================
# STAGE 2: Final Runtime
# ==========================================
FROM python:3.10-slim-trixie

EXPOSE 5000
WORKDIR /base

# Install runtime locales and clean up immediately
RUN apt-get update && apt-get install -y --no-install-recommends locales \
    && printf '%s\n%s\n' 'en_US.UTF-8 UTF-8' 'ru_RU.UTF-8 UTF-8' >> /etc/locale.gen \
    && locale-gen \
    && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
    && rm -rf /var/lib/apt/lists/*

ENV LANG='en_US.UTF-8' \
    LANGUAGE='en_US.UTF-8' \
    LC_ALL='en_US.UTF-8' \
    NLTK_DATA='/usr/local/share/nltk_data' \
    PATH="/venv/bin:$PATH"

# Copy the compiled virtual environment, NLTK data, and the downloaded model weights
COPY --from=builder /venv /venv
COPY --from=builder /usr/local/share/nltk_data /usr/local/share/nltk_data
COPY --from=builder /root/.deeppavlov /root/.deeppavlov

# Create required directories and copy configuration
RUN mkdir /base/train
COPY config/train_tfidf_logreg_en_faq.json /base/config/train_tfidf_logreg_en_faq.json

# Shell form CMD is retained here to allow evaluation of the $LHC_API variable at runtime
CMD ["sh", "-c", "python -m deeppavlov train /base/config/train_tfidf_logreg_en_faq.json && python -m deeppavlov riseapi \"/base/config/$LHC_API\""]
