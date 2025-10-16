# trial-codes


from dotenv import load_dotenv
import os
import docx
from PIL import Image
import base64
import io
import requests
from ultralytics import YOLO
import zipfile
from xml.etree import ElementTree as ET

# Load environment variables
load_dotenv()

VISION_MODEL_URI = os.getenv("LLM_VISION_SERVICE_URL", "")
MODEL_NAME = os.getenv("LLM_VISION_MODEL_NAME", "llama")

# Initialize YOLO model
model = YOLO('./model/yolov11s_best.pt')


def encode_image_to_base64(img):
    """Converts a PIL image to a base64-encoded string."""
    buffered = io.BytesIO()
    img.save(buffered, format="PNG")
    return base64.b64encode(buffered.getvalue()).decode()


def extract_metadata(docx_path):
    """Extracts metadata from the .docx file."""
    doc = docx.Document(docx_path)
    core_props = doc.core_properties
    metadata = {
        "title": core_props.title,
        "author": core_props.author,
        "created": core_props.created.strftime("%Y-%m-%d %H:%M:%S") if core_props.created else None,
        "modified": core_props.modified.strftime("%Y-%m-%d %H:%M:%S") if core_props.modified else None,
        "subject": core_props.subject,
        "keywords": core_props.keywords,
        "category": core_props.category,
        "comments": core_props.comments,
    }
    return metadata


def extract_text(doc):
    """Extracts text from the .docx file."""
    text_md = []
    for para in doc.paragraphs:
        if para.text.strip():
            text_md.append(para.text.strip())
    return text_md


def extract_tables(doc):
    """Extracts tables from the .docx file and formats them in Markdown."""
    tables_md = []
    for table in doc.tables:
        rows = []
        for row in table.rows:
            cells = [cell.text.replace('\n', '') for cell in row.cells]
            rows.append('|' + '|'.join(cells) + '|')

        # Add header separator if table has more than one row
        if len(rows) > 1:
            header_sep = '|' + '|'.join(['---'] * len(table.rows[0].cells)) + '|'
            rows.insert(1, header_sep)

        tables_md.append('\n'.join(rows))
    return tables_md


def extract_images_and_describe_ordered(doc, images_dir):
    """
    Extracts meaningful images from the .docx file and generates descriptions using the vision model.
    Skips small icons/lines. Returns Markdown image entries.
    """
    image_md = []
    image_count = 0

    if not os.path.exists(images_dir):
        os.makedirs(images_dir)

    for para in doc.paragraphs:
        for run in para.runs:
            drawing = run._element.find('.//{http://schemas.openxmlformats.org/drawingml/2006/main}blip')
            if drawing is not None:
                rId = drawing.attrib.get('{http://schemas.openxmlformats.org/officeDocument/2006/relationships}embed')
                if rId and rId in doc.part.rels:
                    rel_obj = doc.part.rels[rId]
                    img_ext = os.path.splitext(rel_obj.target_ref)[1]
                    img_name = f"image_{image_count}{img_ext}"
                    img_path = os.path.join(images_dir, img_name)

                    with open(img_path, 'wb') as img_file:
                        img_file.write(rel_obj.target_part.blob)

                    # Skip small icons
                    try:
                        with Image.open(img_path) as img:
                            width, height = img.size
                            if width < 50 or height < 50:
                                img.close()
                                os.remove(img_path)
                                continue
                    except Exception:
                        if os.path.exists(img_path):
                            try:
                                os.remove(img_path)
                            except PermissionError:
                                pass
                        continue

                    # Describe image via Vision Model
                    try:
                        with open(img_path, "rb") as image_file:
                            encoded = base64.b64encode(image_file.read()).decode()
                            response = requests.post(
                                VISION_MODEL_URI,
                                headers={"Content-Type": "application/json"},
                                json={
                                    "model": MODEL_NAME,
                                    "messages": [
                                        {
                                            "role": "user",
                                            "content": [
                                                {"type": "input_text", "text": "Describe this image. Include any visible text."},
                                                {"type": "input_image", "image_data": encoded}
                                            ]
                                        }
                                    ]
                                }
                            )
                            if response.ok:
                                result = response.json()
                                description = result.get("message", "No description available.")
                            else:
                                description = "Description failed."
                    except Exception as e:
                        description = f"Error describing image: {e}"

                    image_md.append(f"![Image {image_count}]({img_path})\n\n*{description}*")
                    image_count += 1

    return image_md


def extract_flowchart_text(docx_path):
    """
    Extracts text content from shapes, SmartArt, and flowchart elements in DOCX.
    Returns list of detected flowchart/diagram texts.
    """
    flow_texts = []
    ns = {
        'w': 'http://schemas.openxmlformats.org/wordprocessingml/2006/main',
        'a': 'http://schemas.openxmlformats.org/drawingml/2006/main',
        'wps': 'http://schemas.microsoft.com/office/word/2010/wordprocessingShape',
        'v': 'urn:schemas-microsoft-com:vml',
    }

    with zipfile.ZipFile(docx_path, 'r') as docx_zip:
        # Main document.xml
        xml_content = docx_zip.read('word/document.xml')
        tree = ET.fromstring(xml_content)

        for elem in tree.iter():
            tag = elem.tag.split('}')[-1]
            if tag in ['txbx', 'textbox', 'sp', 'shape']:
                texts = []
                for text_elem in elem.findall('.//a:t', ns) + elem.findall('.//w:t', ns):
                    if text_elem.text and text_elem.text.strip():
                        texts.append(text_elem.text.strip())
                if texts:
                    flow_texts.append(" ".join(texts))

        # Also check for /word/drawings/ XMLs
        for name in docx_zip.namelist():
            if name.startswith("word/drawings/") and name.endswith(".xml"):
                drawing_xml = docx_zip.read(name)
                drawing_tree = ET.fromstring(drawing_xml)
                for elem in drawing_tree.iter():
                    tag = elem.tag.split('}')[-1]
                    if tag in ['txbx', 'textbox', 'sp', 'shape']:
                        texts = []
                        for text_elem in elem.findall('.//a:t', ns) + elem.findall('.//w:t', ns):
                            if text_elem.text and text_elem.text.strip():
                                texts.append(text_elem.text.strip())
                        if texts:
                            flow_texts.append(" ".join(texts))

    flow_texts = list(dict.fromkeys(flow_texts))
    return flow_texts if flow_texts else ["_No flowchart or diagram text detected._"]


def create_ordered_md(doc, text_md, tables_md, images_md, images_dir, flow_texts):
    """Creates ordered Markdown with flowchart text appended."""
    ordered_md = []
    text_idx, table_idx, image_idx = 0, 0, 0
    valid_images = set(os.listdir(images_dir)) if os.path.exists(images_dir) else set()
    used_images = set()

    for block in doc.element.body:
        if block.tag.endswith('p'):
            para = docx.text.paragraph.Paragraph(block, doc)
            text = para.text.strip()
            if text and text_idx < len(text_md) and text_md[text_idx] == text:
                ordered_md.append(text_md[text_idx])
                text_idx += 1

            for run in para.runs:
                if 'graphic' in run.element.xml:
                    drawing = run._element.find('.//{http://schemas.openxmlformats.org/drawingml/2006/main}blip')
                    if drawing is not None:
                        rId = drawing.attrib.get('{http://schemas.openxmlformats.org/officeDocument/2006/relationships}embed')
                        if rId and rId in doc.part.rels:
                            rel_obj = doc.part.rels[rId]
                            img_ext = os.path.splitext(rel_obj.target_ref)[1]
                            img_name = f"image_{len(used_images)}{img_ext}"
                            if img_name in valid_images and image_idx < len(images_md):
                                ordered_md.append(images_md[image_idx])
                                used_images.add(img_name)
                                image_idx += 1

        elif block.tag.endswith('tbl'):
            if table_idx < len(tables_md):
                ordered_md.append(tables_md[table_idx])
                table_idx += 1

    # Add flowchart/diagram texts at the end
    ordered_md.append("## Flowchart / Diagram Texts")
    ordered_md.extend(flow_texts)

    return ordered_md


def save_metadata(metadata, meta_path):
    """Saves metadata to a text file."""
    with open(meta_path, "w", encoding="utf-8") as f:
        for key, value in metadata.items():
            f.write(f"{key}: {value}\n")


def save_to_md(content_list, md_path):
    """Saves content to a Markdown file."""
    with open(md_path, "w", encoding="utf-8") as f:
        for item in content_list:
            f.write(item + "\n\n")


if __name__ == "__main__":
    input_folder = os.getenv("INPUT_FOLDER", "input_docs")
    fallback_folder = "input"
    output_folder = os.getenv("OUTPUT_FOLDER", "output_results")

    if not os.path.exists(output_folder):
        os.makedirs(output_folder)

    if not os.path.exists(input_folder):
        if os.path.exists(fallback_folder):
            input_folder = fallback_folder
            print(f"Using existing input folder: '{input_folder}'")
        else:
            os.makedirs(input_folder)
            print(f"Created input folder: '{input_folder}'. Please add .docx files and re-run the script.")

    for filename in os.listdir(input_folder):
        if filename.lower().endswith(".docx"):
            docx_file = os.path.join(input_folder, filename)
            base_name = os.path.splitext(filename)[0]

            images_dir = os.path.join(output_folder, f"{base_name}_images")
            md_file = os.path.join(output_folder, f"{base_name}_content.md")
            meta_file = os.path.join(output_folder, f"{base_name}_metadata.txt")

            doc = docx.Document(docx_file)

            metadata = extract_metadata(docx_file)
            save_metadata(metadata, meta_file)

            text_md = extract_text(doc)
            tables_md = extract_tables(doc)
            images_md = extract_images_and_describe_ordered(doc, images_dir)
            flow_texts = extract_flowchart_text(docx_file)

            ordered_md = create_ordered_md(doc, text_md, tables_md, images_md, images_dir, flow_texts)
            save_to_md(ordered_md, md_file)

            print(f"Processed: {filename}")
            print(f"Metadata saved to {meta_file}")
            print(f"Images saved in {images_dir}")
            print(f"Markdown content saved to {md_file}")
