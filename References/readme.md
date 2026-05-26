# 9. References

References are organized into six categories: academic papers, technical reports, books and textbooks, software libraries and SDKs, online resources and documentation, and related open-source projects. Within each category, entries are listed alphabetically by first author or project name. Citations follow an IEEE-numeric style for inline use throughout the report.

---

## 9.1 Academic Papers

[1] T. B. Brown, B. Mann, N. Ryder, M. Subbiah, J. Kaplan, P. Dhariwal, A. Neelakantan, P. Shyam, G. Sastry, A. Askell, S. Agarwal, A. Herbert-Voss, G. Krueger, T. Henighan, R. Child, A. Ramesh, D. M. Ziegler, J. Wu, C. Winter, C. Hesse, M. Chen, E. Sigler, M. Litwin, S. Gray, B. Chess, J. Clark, C. Berner, S. McCandlish, A. Radford, I. Sutskever, and D. Amodei, **"Language Models are Few-Shot Learners,"** in *Advances in Neural Information Processing Systems (NeurIPS)*, 2020. (The GPT-3 paper introducing in-context learning — the basis for the example-rich planner prompt in `agent/planner.py`.)

[2] J. Devlin, M.-W. Chang, K. Lee, and K. Toutanova, **"BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding,"** in *Proceedings of NAACL-HLT*, 2019. (Foundational pre-training work cited for general background on transformer-based language understanding.)

[3] A. Dosovitskiy, L. Beyer, A. Kolesnikov, D. Weissenborn, X. Zhai, T. Unterthiner, M. Dehghani, M. Minderer, G. Heigold, S. Gelly, J. Uszkoreit, and N. Houlsby, **"An Image is Worth 16x16 Words: Transformers for Image Recognition at Scale,"** in *Proceedings of ICLR*, 2021. (Background for vision transformers underlying modern multimodal models like Gemini Vision.)

[4] Google DeepMind, **"Gemini 2.5: Pushing the Frontier with Advanced Multimodal Reasoning, Long Context, and Agentic Capabilities,"** *Technical Report*, Google DeepMind, 2025. (The underlying model family used throughout R.A.Y.A.)

[5] Google DeepMind, **"Gemini: A Family of Highly Capable Multimodal Models,"** *Technical Report*, Google, 2023. (Predecessor report providing baseline multimodal architecture context.)

[6] C. Packer, V. Fang, S. G. Patil, K. Lin, S. Wooders, and J. E. Gonzalez, **"MemGPT: Towards LLMs as Operating Systems,"** *arXiv preprint arXiv:2310.08560*, 2023. (Hierarchical memory architecture that informed the design of R.A.Y.A's six-category long-term memory.)

[7] T. Schick, J. Dwivedi-Yu, R. Dessì, R. Raileanu, M. Lomeli, L. Zettlemoyer, N. Cancedda, and T. Scialom, **"Toolformer: Language Models Can Teach Themselves to Use Tools,"** in *Advances in Neural Information Processing Systems (NeurIPS)*, 2023. (Foundational paper on tool-augmented LLMs — direct ancestor of modern function-calling.)

[8] N. Shinn, F. Cassano, E. Berman, A. Gopinath, K. Narasimhan, and S. Yao, **"Reflexion: Language Agents with Verbal Reinforcement Learning,"** in *Advances in Neural Information Processing Systems (NeurIPS)*, 2023. (Self-criticism loop concept underlying the design of `agent/error_handler.py`.)

[9] R. Sutton and A. G. Barto, **"Reinforcement Learning: An Introduction,"** *2nd ed., MIT Press*, 2018. (Cited as background for the agent retry/replan/abort decision framework.)

[10] A. Vaswani, N. Shazeer, N. Parmar, J. Uszkoreit, L. Jones, A. N. Gomez, Ł. Kaiser, and I. Polosukhin, **"Attention Is All You Need,"** in *Advances in Neural Information Processing Systems (NeurIPS)*, 2017. (Foundational Transformer paper underlying every modern LLM.)

[11] S. Yao, J. Zhao, D. Yu, N. Du, I. Shafran, K. Narasimhan, and Y. Cao, **"ReAct: Synergizing Reasoning and Acting in Language Models,"** in *Proceedings of ICLR*, 2023. (The reason-then-act loop that R.A.Y.A's main Live conversation pattern approximates.)

---

## 9.2 Technical Reports and White Papers

[12] Anthropic, **"Introducing Computer Use, a New Claude 3.5 Sonnet, and Claude 3.5 Haiku,"** *Anthropic Technical Announcement*, October 2024. (The first widely-publicized "computer-using agent," directly relevant to R.A.Y.A's `computer_control` tool design.)

[13] Google, **"Gemini API: Live API Documentation,"** *Google AI Developer Documentation*, 2025. URL: https://ai.google.dev/gemini-api/docs/live (Primary reference for the streaming voice protocol used in `main.py`.)

[14] OpenAI, **"Introducing Operator,"** *OpenAI Technical Announcement*, January 2025. (Browser-based computer-using agent — comparable system referenced in Section 2.2.)

[15] Google, **"Project Mariner: Exploring the Future of Browser-Based Agents,"** *Google Labs Announcement*, December 2024. (Direct predecessor to many design decisions in R.A.Y.A's `browser_control` tool.)

---

## 9.3 Books and Textbooks

[16] I. Goodfellow, Y. Bengio, and A. Courville, **"Deep Learning,"** *MIT Press*, 2016. (General reference for neural network foundations.)

[17] D. Jurafsky and J. H. Martin, **"Speech and Language Processing,"** *3rd ed. (draft), Stanford University*, 2024. (Reference for speech recognition and natural language processing background.)

[18] M. Lutz, **"Learning Python,"** *5th ed., O'Reilly Media*, 2013. (Python language reference.)

[19] L. Ramalho, **"Fluent Python,"** *2nd ed., O'Reilly Media*, 2022. (Advanced Python idioms, asyncio patterns used in `main.py`.)

[20] S. Russell and P. Norvig, **"Artificial Intelligence: A Modern Approach,"** *4th ed., Pearson*, 2020. (Cited as background for the planner-executor agent architecture.)

[21] B. Slatkin, **"Effective Python,"** *2nd ed., Addison-Wesley*, 2019. (Best-practices reference for Python application design.)

---

## 9.4 Software Libraries and SDKs

[22] Anthropic, **"Anthropic Python SDK,"** PyPI package `anthropic`, 2024. URL: https://github.com/anthropics/anthropic-sdk-python

[23] **BeautifulSoup4**, PyPI package `beautifulsoup4`, Crummy.com. URL: https://www.crummy.com/software/BeautifulSoup/

[24] **comtypes**, Python COM bridge library, PyPI package `comtypes`. URL: https://github.com/enthought/comtypes

[25] **duckduckgo-search**, Python wrapper for DuckDuckGo search, PyPI package `duckduckgo-search`.

[26] Google, **"google-genai SDK,"** PyPI package `google-genai`, Google AI, 2025. URL: https://github.com/googleapis/python-genai

[27] Google, **"google-generativeai SDK,"** PyPI package `google-generativeai`, Google AI. URL: https://github.com/google-gemini/generative-ai-python

[28] **Inno Setup**, J. Russell, 2025. URL: https://jrsoftware.org/isinfo.php (Windows installer authoring tool used in `scripts/build_windows.ps1`.)

[29] **Microsoft Playwright**, Microsoft, 2024. URL: https://playwright.dev/python (Cross-browser automation engine for `actions/browser_control.py`.)

[30] **mss**, ultra-fast cross-platform screen capture, PyPI package `mss`. URL: https://github.com/BoboTiG/python-mss

[31] **NumPy**, C. R. Harris et al., **"Array programming with NumPy,"** *Nature*, vol. 585, pp. 357–362, 2020. URL: https://numpy.org

[32] **OpenAI Python SDK**, PyPI package `openai`. URL: https://github.com/openai/openai-python (Referenced for comparison; not used in R.A.Y.A.)

[33] **OpenCV-Python**, OpenCV.org. URL: https://opencv.org

[34] **Pillow (PIL Fork)**, A. Clark and Contributors. URL: https://python-pillow.org

[35] **psutil**, G. Rodola. URL: https://github.com/giampaolo/psutil (Process and system utilities.)

[36] **pyautogui**, A. Sweigart. URL: https://github.com/asweigart/pyautogui

[37] **pycaw**, Python Core Audio Windows Library. URL: https://github.com/AndreMiras/pycaw

[38] **PyInstaller**, PyInstaller Development Team. URL: https://pyinstaller.org

[39] **pyperclip**, A. Sweigart. URL: https://github.com/asweigart/pyperclip

[40] **pywinauto**, M. Klyuyev. URL: https://github.com/pywinauto/pywinauto

[41] **requests**, K. Reitz. URL: https://requests.readthedocs.io

[42] **send2trash**, V. Légaré. URL: https://github.com/arsenetar/send2trash

[43] **sounddevice**, M. Geier, PyPI package `sounddevice`. URL: https://python-sounddevice.readthedocs.io (Audio I/O over PortAudio.)

[44] **Tkinter**, Python Standard Library, Python Software Foundation. URL: https://docs.python.org/3/library/tkinter.html

[45] **win10toast**, J. Clayton. URL: https://github.com/jithurjacob/Windows-10-Toast-Notifications

[46] **youtube-transcript-api**, J. Depoix. URL: https://github.com/jdepoix/youtube-transcript-api

---

## 9.5 Online Resources and Documentation

[47] **Python Software Foundation**, **"Python 3.11 Documentation — asyncio,"** 2024. URL: https://docs.python.org/3.11/library/asyncio.html

[48] **Python Software Foundation**, **"Python 3.11 Documentation — threading,"** 2024. URL: https://docs.python.org/3.11/library/threading.html

[49] **Python Software Foundation**, **"What's New in Python 3.11 — asyncio.TaskGroup,"** 2022. URL: https://docs.python.org/3/whatsnew/3.11.html

[50] **PortAudio**, R. Bencina and P. Burk, **"PortAudio Cross-Platform Audio I/O Library,"** 2025. URL: https://www.portaudio.com

[51] **Conda**, Anaconda Inc., 2025. URL: https://docs.conda.io

[52] **Microsoft**, **"Windows Core Audio APIs,"** Microsoft Learn, 2024. URL: https://learn.microsoft.com/windows/win32/coreaudio/

[53] **Microsoft**, **"Microsoft UI Automation,"** Microsoft Learn, 2024. URL: https://learn.microsoft.com/windows/win32/winauto/entry-uiauto-win32

[54] **Steam**, Valve Corporation, 2025. URL: https://store.steampowered.com (Referenced as integration target for `game_updater`.)

[55] **Epic Games Launcher**, Epic Games Inc., 2025. URL: https://store.epicgames.com (Referenced as integration target for `game_updater`.)

[56] **DuckDuckGo**, 2025. URL: https://duckduckgo.com (Search backend for `web_search`.)

[57] **YouTube Data API**, Google, 2025. URL: https://developers.google.com/youtube

---

## 9.6 Related Open-Source Projects and Frameworks

[58] **LangChain**, H. Chase and contributors, 2022 onwards. URL: https://github.com/langchain-ai/langchain (Reference framework for LLM tool-calling; design philosophy contrasted in Section 2.2.)

[59] **LangGraph**, LangChain Inc., 2024. URL: https://github.com/langchain-ai/langgraph (Graph-based agent framework.)

[60] **AutoGPT**, Significant-Gravitas, 2023. URL: https://github.com/Significant-Gravitas/AutoGPT (Influential early autonomous agent.)

[61] **BabyAGI**, Y. Nakajima, 2023. URL: https://github.com/yoheinakajima/babyagi (Recursive task-decomposition agent prototype.)

[62] **CrewAI**, CrewAI Inc., 2024. URL: https://github.com/crewAIInc/crewAI (Multi-agent framework.)

[63] **AutoGen**, Microsoft Research, 2023. URL: https://github.com/microsoft/autogen (Conversational multi-agent framework.)

[64] **Mycroft AI**, Mycroft AI Inc. (predecessor), now OpenVoiceOS. URL: https://github.com/OpenVoiceOS/ovos-core (Open-source voice assistant with skill-folder architecture.)

[65] **Leon AI**, L. Bourichi, 2019 onwards. URL: https://github.com/leon-ai/leon (Open-source personal assistant in Node.js/Python.)

[66] **Rhasspy**, M. Hansen, 2018 onwards. URL: https://github.com/rhasspy/rhasspy (Fully offline open-source voice assistant.)

[67] **Jarvis**, Sukeesh, 2018 onwards. URL: https://github.com/sukeesh/Jarvis (Pre-LLM Python personal assistant project.)

[68] **Pipecat**, Daily.co, 2024 onwards. URL: https://github.com/pipecat-ai/pipecat (Open-source framework for streaming voice agents.)

[69] **LiveKit Agents**, LiveKit Inc., 2024 onwards. URL: https://github.com/livekit/agents (Real-time voice agent framework.)

[70] **Whisper**, OpenAI, 2022. URL: https://github.com/openai/whisper (Open-source speech recognition model; comparative reference.)

[71] **Coqui TTS**, Coqui AI, 2021 onwards. URL: https://github.com/coqui-ai/TTS (Open-source text-to-speech engine; comparative reference for future on-device fallback.)

[72] **Home Assistant**, Home Assistant Community, 2013 onwards. URL: https://www.home-assistant.io (Referenced in future-scope smart-home integration plans.)

---

## 9.7 Citation Notes

- All URLs were accessible as of the date of report writing. Should any URL be dead at the time of review, the corresponding GitHub mirrors or Wayback Machine snapshots should locate the canonical content.
- arXiv preprints are cited in their canonical form. The published proceedings version (where one exists) is cited in preference to the preprint when both are available.
- Where a library has no clearly authoritative academic publication, the GitHub repository URL is used as the primary reference.
- Software versions are not pinned in citations; users should consult `requirements.txt` and `environment.yml` in the project root for the exact versions used in the R.A.Y.A v2.4 build.

---

## 9.8 Acknowledgement of Tools Used in Preparation of This Report

In the spirit of full transparency, the following tools were used in the preparation of this report:

- **VS Code** as the primary editor.
- **Claude Code (Anthropic CLI)** for assistance with drafting, organizing, and refining the report's prose.
- **Microsoft PowerShell 7+** as the development shell.
- **Git** for version control of the report alongside the codebase.

The technical content, architectural decisions, code, and evaluation findings are the developer's own work. AI assistance was used in the same way a writer might use spell-check, grammar tools, or a thoughtful editor — to refine expression, not to generate technical substance.

---

End of References.
