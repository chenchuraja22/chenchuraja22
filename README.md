- 👋 Hi, I’m @chenchuraja22
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...

<!---
chenchuraja22/chenchuraja22 is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->

TABLES : dd03l.
TYPES:
BEGIN OF ty_s_cline,
  LINE TYPE C LENGTH 1024, " Line of type Character
END OF ty_s_cline .

DATA: gr_table     TYPE REF TO cl_salv_table,          " Basis Class for Simple Tables
      gt_outtab    TYPE STANDARD TABLE OF sflight,     " Flight
      lr_functions TYPE REF TO cl_salv_functions_list, " Generic and User-Defined Functions in List-Type Tables
      lr_layout    TYPE REF TO cl_salv_layout,         " Settings for Layout
      lv_xml_type  TYPE salv_bs_constant,
      lv_xml       TYPE xstring,
      gt_srctab    TYPE STANDARD TABLE OF ty_s_cline,
      gt_len       TYPE I,                             " Len of type Integers
      lv_file_xlsx TYPE string,
      lv_file      TYPE localfile,                     " Local file for upload/download
      ls_key       TYPE salv_s_layout_key,             " Layout Key
      lr_content   TYPE REF TO cl_salv_form_element.   " General Element in Design Object

*PARAMETERS table TYPE tabname16  DEFAULT 'SCARR'.

SELECT-OPTIONS : table FOR dd03l-tabname NO INTERVALS.

DATA go_structdesc TYPE REF TO cl_abap_structdescr.

DATA go_error TYPE REF TO cx_root.

DATA gv_error TYPE string.

DATA gt_field TYPE ddfields.

DATA gt_select TYPE TABLE OF char72.

DATA gs_field  TYPE dfies.

DATA: lt_components  TYPE abap_component_tab,    "Selected Fields
      ls_component   LIKE LINE OF lt_components,
      lo_datadescr   TYPE REF TO cl_abap_datadescr,
      lo_structdescr TYPE REF TO cl_abap_structdescr,
      lo_tabledescr  TYPE REF TO cl_abap_tabledescr,
      lr_table       TYPE REF TO data.

DATA: gr_alv TYPE REF TO cl_salv_table.

FIELD-SYMBOLS <gt_table> TYPE STANDARD TABLE.  "Better than Any because Alv Displays Std TAb Only


START-OF-SELECTION.

  LOOP AT table[] INTO DATA(ls_tab) .

    DATA : lv_table TYPE tabname16.

    CLEAR: table,gt_field,lt_components,lo_structdescr, lo_tabledescr,gt_select.

    lv_table = ls_tab-low.
    TRY.
        go_structdesc ?= cl_abap_structdescr=>describe_by_name( lv_table ).
        gt_field = go_structdesc->get_ddic_field_list(  ).

      CATCH cx_root INTO go_error.
        gv_error = go_error->get_text( ).
        MESSAGE gv_error TYPE 'I'.

    ENDTRY.


* Build op Components

    LOOP AT gt_field INTO gs_field.

      lo_datadescr ?=  cl_abap_datadescr=>describe_by_name( gs_field-rollname ).

      ls_component-name = gs_field-fieldname.

      ls_component-type = lo_datadescr.

      APPEND ls_component TO lt_components.

    ENDLOOP.

* Create structdescr

    lo_structdescr = cl_abap_structdescr=>create( lt_components ).

* Create table descr

    lo_tabledescr =  cl_abap_tabledescr=>create( lo_structdescr ).  "Create has more param's that are interesting..

* Create Internal table mbv Our dynamically Created Table Type

    CREATE DATA lr_table TYPE HANDLE lo_tabledescr.

* Assign Field Symbol

    ASSIGN lr_table->* TO <gt_table>.

    LOOP AT lt_components INTO ls_component.

      APPEND ls_component-name TO gt_select.

    ENDLOOP.

    SELECT (gt_select) FROM (lv_table) INTO CORRESPONDING FIELDS OF TABLE <gt_table>.


  TRY.

    cl_salv_table=>factory(
    IMPORTING
      r_salv_table = gr_table
    CHANGING
      t_table      = <gt_table> ).

  CATCH cx_salv_msg. "#EC NO_HANDLER

  ENDTRY.

  lr_functions = gr_table->get_functions( ).

  lr_functions->set_all( abap_true ).

  lr_layout = gr_table->get_layout( ).

  ls_key-REPORT = sy-repid.

  lr_layout->set_key( ls_key ).

  lr_layout->set_save_restriction( if_salv_c_layout=>restrict_user_independant ).

  gr_table->set_top_of_list( lr_content ).

  gr_table->set_end_of_list( lr_content ).

*  gr_table->display( ).

  lv_xml_type =  if_salv_bs_xml=>c_type_xlsx. "if_salv_bs_xml=>c_type_mhtml.

  lv_xml      = gr_table->to_xml( xml_type = lv_xml_type ).

  CALL FUNCTION 'SCMS_XSTRING_TO_BINARY'
  EXPORTING
    BUFFER        = lv_xml
*     APPEND_TO_TABLE       = ' '
  IMPORTING
    output_length = gt_len
  TABLES
    binary_tab    = gt_srctab.

*lv_file_xlsx = 'filepath/file.xlsx'.

  CALL METHOD cl_gui_frontend_services=>gui_download
  EXPORTING
    bin_filesize            = gt_len
    filename                = lv_file_xlsx
    filetype                = 'BIN'
  CHANGING
    data_tab                = gt_srctab
  EXCEPTIONS
    file_write_error        = 1
    no_batch                = 2
    gui_refuse_filetransfer = 3
    invalid_type            = 4
    no_authority            = 5
    unknown_error           = 6
    header_not_allowed      = 7
    separator_not_allowed   = 8
    filesize_not_allowed    = 9
    header_too_long         = 10
    dp_error_create         = 11
    dp_error_send           = 12
    dp_error_write          = 13
    unknown_dp_error        = 14
    access_denied           = 15
    dp_out_of_memory        = 16
    disk_full               = 17
    dp_timeout              = 18
    file_not_found          = 19
    dataprovider_exception  = 20
    control_flush_error     = 21
    not_supported_by_gui    = 22
    error_no_gui            = 23
    OTHERS                  = 24.

  IF sy-subrc <> 0.

    MESSAGE ID sy-msgid TYPE sy-msgty NUMBER sy-msgno
    WITH sy-msgv1 sy-msgv2 sy-msgv3 sy-msgv4.
  ENDIF. " IF sy-subrc <> 0

CLEAR : ls_tab.
  ENDLOOP.
