# ==========================================
# STAGE 1: Builder
# ==========================================
FROM python:3.10-slim-trixie AS builder

ARG COMMIT=6e1036dbfcde1c293b50c742f0736a3965dd1e0d

RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential \
        git \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /base

RUN python -m venv /venv
ENV PATH="/venv/bin:$PATH"

RUN git clone https://github.com/deepmipt/DeepPavlov.git \
    && cd DeepPavlov \
    && git checkout $COMMIT \
    && pip install --no-cache-dir . \
    && cd /base && rm -rf DeepPavlov

RUN pip install --no-cache-dir email-validator lxml

COPY config/train_tfidf_logreg_en_faq.json /base/config/

RUN python -m deeppavlov install /base/config/train_tfidf_logreg_en_faq.json \
    && pip install --no-cache-dir "scikit-learn==1.0.2" "scipy<1.10.0" "numpy<1.24.0" \
    && python -m deeppavlov download /base/config/train_tfidf_logreg_en_faq.json

# Pre-download NLTK packages to the system-wide global directory
RUN python -m nltk.downloader -d /usr/share/nltk_data \
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

# 1. Install locales and create the non-root 'dpuser' with a home directory
RUN apt-get update && apt-get install -y --no-install-recommends locales \
    && printf '%s\n%s\n' 'en_US.UTF-8 UTF-8' 'ru_RU.UTF-8 UTF-8' >> /etc/locale.gen \
    && locale-gen \
    && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
    && rm -rf /var/lib/apt/lists/* \
    && groupadd -r dpuser && useradd -r -g dpuser -m -d /home/dpuser dpuser

ENV LANG='en_US.UTF-8' \
    LANGUAGE='en_US.UTF-8' \
    LC_ALL='en_US.UTF-8' \
    NLTK_DATA='/usr/share/nltk_data' \
    PATH="/venv/bin:$PATH"

# 2. Copy the venv and change ownership to dpuser
COPY --from=builder --chown=dpuser:dpuser /venv /venv
COPY --from=builder /venv /venv
COPY --from=builder /usr/share/nltk_data /usr/share/nltk_data

# 3. Copy NLTK and DeepPavlov data directly into dpuser's home directory
COPY --from=builder /usr/share/nltk_data /usr/share/nltk_data
COPY --from=builder --chown=dpuser:dpuser /root/.deeppavlov /home/dpuser/.deeppavlov

# 4. Create working directories and explicitly grant ownership to dpuser
RUN mkdir -p /base/train /base/config /base/deep \
    && chown -R dpuser:dpuser /base

# 5. Copy the config file under dpuser ownership
COPY --chown=dpuser:dpuser config/train_tfidf_logreg_en_faq.json /base/config/train_tfidf_logreg_en_faq.json

# 6. Switch to the non-root user before running any application code
USER dpuser

CMD ["sh", "-c", "python -m deeppavlov train /base/config/train_tfidf_logreg_en_faq.json && python -m deeppavlov riseapi \"/base/config/$LHC_API\""]
