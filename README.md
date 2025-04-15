import sys
import json
from datetime import datetime
from PyQt5.QtWidgets import (
    QApplication, QMainWindow, QWidget, QVBoxLayout, QHBoxLayout,
    QPushButton, QLabel, QLineEdit, QComboBox, QTableWidget,
    QTableWidgetItem, QHeaderView, QInputDialog, QFileDialog,
    QTabWidget, QFormLayout, QDialog, QMessageBox, QTextEdit,
    QListWidget, QListWidgetItem, QCheckBox
)
from PyQt5.QtCore import Qt
from PyQt5.QtChart import QChart, QChartView, QBarSet, QBarSeries, QValueAxis, QBarCategoryAxis
from PyQt5.QtGui import QPainter
import pytz
from hijri_converter import Hijri, Gregorian

# ------------------- تابع تبدیل تاریخ -------------------
def get_jalali_datetime():
    tz = pytz.timezone('Asia/Tehran')
    ir_time = datetime.now(tz)
    g = Gregorian(ir_time.year, ir_time.month, ir_time.day)
    j = g.to_hijri()
    jalali_date = f"{j.year-42}/{j.month-9}/{j.day+9}"
    time_str = ir_time.strftime("%H:%M")
    return f"{jalali_date} ساعت {time_str}"

# ------------------- دیالوگ مدیریت مشتریان -------------------
class CustomerDialog(QDialog):
    def __init__(self, parent=None, customer=None):
        super().__init__(parent)
        self.setWindowTitle("مدیریت مشتریان" if not customer else "ویرایش مشتری")
        self.setGeometry(200, 200, 400, 300)
        
        layout = QVBoxLayout()
        
        form_layout = QFormLayout()
        self.txt_name = QLineEdit()
        self.txt_phone = QLineEdit()
        self.txt_address = QTextEdit()
        self.txt_address.setMaximumHeight(100)
        
        form_layout.addRow("نام مشتری:", self.txt_name)
        form_layout.addRow("تلفن:", self.txt_phone)
        form_layout.addRow("آدرس:", self.txt_address)
        
        if customer:
            self.txt_name.setText(customer.get('name', ''))
            self.txt_phone.setText(customer.get('phone', ''))
            self.txt_address.setText(customer.get('address', ''))
        
        btn_layout = QHBoxLayout()
        self.btn_save = QPushButton("ذخیره")
        self.btn_cancel = QPushButton("انصراف")
        
        btn_layout.addWidget(self.btn_save)
        btn_layout.addWidget(self.btn_cancel)
        
        layout.addLayout(form_layout)
        layout.addLayout(btn_layout)
        
        self.setLayout(layout)
        
        self.btn_save.clicked.connect(self.accept)
        self.btn_cancel.clicked.connect(self.reject)
    
    def get_customer_data(self):
        return {
            'name': self.txt_name.text().strip(),
            'phone': self.txt_phone.text().strip(),
            'address': self.txt_address.toPlainText().strip()
        }

# ------------------- دیالوگ ابزار زدن -------------------
class ToolingDialog(QDialog):
    def __init__(self, parent=None):
        super().__init__(parent)
        self.setWindowTitle("ابزار زدن")
        self.setGeometry(300, 300, 500, 400)
        layout = QVBoxLayout()
        
        customer_layout = QHBoxLayout()
        self.cmb_customer = QComboBox()
        self.btn_add_customer = QPushButton("+")
        self.btn_remove_customer = QPushButton("-")
        
        customer_layout.addWidget(QLabel("نام مشتری:"))
        customer_layout.addWidget(self.cmb_customer)
        customer_layout.addWidget(self.btn_add_customer)
        customer_layout.addWidget(self.btn_remove_customer)
        layout.addLayout(customer_layout)
        
        self.lbl_stone_type = QLabel("نوع سنگ (می‌توانید چند مورد انتخاب کنید):")
        self.list_stone_type = QListWidget()
        self.list_stone_type.setSelectionMode(QListWidget.MultiSelection)
        self.list_stone_type.addItems(["گرانیت", "مرمریت", "تراورتن", "چینی"])
        layout.addWidget(self.lbl_stone_type)
        layout.addWidget(self.list_stone_type)
        
        self.lbl_tool_type = QLabel("نوع ابزار (می‌توانید چند مورد انتخاب کنید):")
        self.list_tool_type = QListWidget()
        self.list_tool_type.setSelectionMode(QListWidget.MultiSelection)
        self.list_tool_type.addItems([
            "2 پله", "3 پله", "نیمه گرد", "مام گرد",
            "خطی یک گرد", "خطی تخت"
        ])
        layout.addWidget(self.lbl_tool_type)
        layout.addWidget(self.list_tool_type)
        
        self.lbl_tool_size = QLabel("سایز ابزار:")
        self.cmb_tool_size = QComboBox()
        self.cmb_tool_size.addItems(["2 سانت", "3 سانت"])
        layout.addWidget(self.lbl_tool_size)
        layout.addWidget(self.cmb_tool_size)
        
        price_layout = QHBoxLayout()
        self.lbl_price = QLabel("قیمت هر متر:")
        self.txt_price = QLineEdit()
        self.lbl_area = QLabel("متراژ:")
        self.txt_area = QLineEdit()
        
        price_layout.addWidget(self.lbl_price)
        price_layout.addWidget(self.txt_price)
        price_layout.addWidget(self.lbl_area)
        price_layout.addWidget(self.txt_area)
        layout.addLayout(price_layout)
        
        self.btn_submit = QPushButton("ثبت ابزار زدن")
        self.btn_submit.clicked.connect(self.submit)
        layout.addWidget(self.btn_submit)
        
        self.setLayout(layout)
        
        self.btn_add_customer.clicked.connect(self.add_customer)
        self.btn_remove_customer.clicked.connect(self.remove_customer)
        self.update_customer_list()
    
    def update_customer_list(self):
        self.cmb_customer.clear()
        if hasattr(self.parent(), 'customers'):
            self.cmb_customer.addItems([c['name'] for c in self.parent().customers])
    
    def add_customer(self):
        dialog = CustomerDialog(self)
        if dialog.exec_() == QDialog.Accepted:
            customer_data = dialog.get_customer_data()
            if customer_data['name']:
                self.parent().customers.append(customer_data)
                self.parent().save_data()
                self.update_customer_list()
                self.cmb_customer.setCurrentText(customer_data['name'])
    
    def remove_customer(self):
        customer_name = self.cmb_customer.currentText()
        if customer_name:
            reply = QMessageBox.question(
                self, "حذف مشتری",
                f"آیا از حذف مشتری '{customer_name}' مطمئن هستید؟",
                QMessageBox.Yes | QMessageBox.No
            )
            if reply == QMessageBox.Yes:
                self.parent().customers = [
                    c for c in self.parent().customers if c['name'] != customer_name
                ]
                self.parent().save_data()
                self.update_customer_list()
    
    def submit(self):
        if not self.cmb_customer.currentText():
            QMessageBox.warning(self, "خطا", "لطفاً مشتری را انتخاب کنید!")
            return
        
        selected_stones = [item.text() for item in self.list_stone_type.selectedItems()]
        if not selected_stones:
            QMessageBox.warning(self, "خطا", "لطفاً حداقل یک نوع سنگ انتخاب کنید!")
            return
        
        selected_tools = [item.text() for item in self.list_tool_type.selectedItems()]
        if not selected_tools:
            QMessageBox.warning(self, "خطا", "لطفاً حداقل یک نوع ابزار انتخاب کنید!")
            return
        
        try:
            price = float(self.txt_price.text())
            area = float(self.txt_area.text())
        except ValueError:
            QMessageBox.warning(self, "خطا", "قیمت و متراژ باید عدد باشند!")
            return
        
        for stone in selected_stones:
            for tool in selected_tools:
                transaction = {
                    "type": "ابزار زدن",
                    "datetime": get_jalali_datetime(),
                    "customer": self.cmb_customer.currentText(),
                    "stone_type": stone,
                    "tool_type": tool,
                    "tool_size": self.cmb_tool_size.currentText(),
                    "price": price,
                    "area": area,
                    "total": price * area,
                    "paid": False,
                    "stone_detail": "",
                    "cut_thickness": "",
                    "cut_type": ""
                }
                self.parent().add_transaction(transaction)
        
        QMessageBox.information(self, "موفق", "تراکنش‌ها با موفقیت ثبت شدند!")
        self.accept()

# ------------------- دیالوگ گزارش مالی -------------------
class ReportDialog(QDialog):
    def __init__(self, report_text, parent=None):
        super().__init__(parent)
        self.setWindowTitle("گزارش مالی")
        self.setGeometry(300, 300, 600, 500)
        
        layout = QVBoxLayout()
        self.text_edit = QTextEdit()
        self.text_edit.setPlainText(report_text)
        self.text_edit.setReadOnly(True)
        layout.addWidget(self.text_edit)
        
        btn_layout = QHBoxLayout()
        self.btn_print = QPushButton("چاپ")
        self.btn_save = QPushButton("ذخیره به فایل")
        self.btn_close = QPushButton("بستن")
        
        btn_layout.addWidget(self.btn_print)
        btn_layout.addWidget(self.btn_save)
        btn_layout.addWidget(self.btn_close)
        layout.addLayout(btn_layout)
        self.setLayout(layout)
        
        self.btn_print.clicked.connect(self.print_report)
        self.btn_save.clicked.connect(self.save_report)
        self.btn_close.clicked.connect(self.close)
    
    def print_report(self):
        QMessageBox.information(self, "چاپ", "گزارش آماده چاپ است.")
    
    def save_report(self):
        file_name, _ = QFileDialog.getSaveFileName(
            self, "ذخیره گزارش", "", "Text Files (*.txt);;PDF Files (*.pdf)"
        )
        if file_name:
            try:
                with open(file_name, 'w', encoding='utf-8') as f:
                    f.write(self.text_edit.toPlainText())
                QMessageBox.information(self, "موفق", "گزارش با موفقیت ذخیره شد.")
            except Exception as e:
                QMessageBox.critical(self, "خطا", f"خطا در ذخیره فایل: {str(e)}")

# ------------------- دیالوگ جستجو -------------------
class SearchDialog(QDialog):
    def __init__(self, parent=None):
        super().__init__(parent)
        self.setWindowTitle("جستجوی پیشرفته")
        self.setGeometry(300, 300, 400, 200)
        
        layout = QVBoxLayout()
        form_layout = QFormLayout()
        self.cmb_field = QComboBox()
        self.cmb_field.addItems([
            "نام مشتری", "نوع سنگ", "آدرس", 
            "نوع ابزار", "وضعیت پرداخت"
        ])
        
        self.txt_search = QLineEdit()
        self.chk_paid = QCheckBox("فقط پرداخت شده‌ها")
        
        form_layout.addRow("فیلد جستجو:", self.cmb_field)
        form_layout.addRow("عبارت جستجو:", self.txt_search)
        form_layout.addRow(self.chk_paid)
        
        btn_layout = QHBoxLayout()
        self.btn_search = QPushButton("جستجو")
        self.btn_cancel = QPushButton("انصراف")
        
        btn_layout.addWidget(self.btn_search)
        btn_layout.addWidget(self.btn_cancel)
        layout.addLayout(form_layout)
        layout.addLayout(btn_layout)
        self.setLayout(layout)
        
        self.btn_search.clicked.connect(self.accept)
        self.btn_cancel.clicked.connect(self.reject)
        self.cmb_field.currentTextChanged.connect(self.update_search_field)
        self.update_search_field()
    
    def update_search_field(self):
        field = self.cmb_field.currentText()
        if field == "وضعیت پرداخت":
            self.txt_search.setEnabled(False)
            self.chk_paid.setEnabled(True)
        else:
            self.txt_search.setEnabled(True)
            self.chk_paid.setEnabled(False)
    
    def get_search_params(self):
        field = self.cmb_field.currentText()
        value = self.txt_search.text().strip() if field != "وضعیت پرداخت" else ""
        paid_only = self.chk_paid.isChecked() if field == "وضعیت پرداخت" else False
        return {
            'field': field,
            'value': value,
            'paid_only': paid_only
        }

# ------------------- برنامه اصلی -------------------
class StoneCuttingApp(QMainWindow):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("برنامه مدیریت فروش و برش سنگ")
        self.setGeometry(100, 100, 1200, 800)
        
        self.current_user = None
        self.transactions = []
        self.customers = []
        self.users = []
        self.inventory = []
        self.logs = []
        
        self.load_data()
        
        if not self.authenticate_user():
            sys.exit()
        
        self.init_ui()
    
    def authenticate_user(self):
        dialog = AuthDialog(self.users, self)
        if dialog.exec_() == QDialog.Accepted:
            self.current_user = dialog.login_username.text()
            self.log_action("ورود کاربر")
            return True
        return False
    
    def log_action(self, action):
        log_entry = f"{get_jalali_datetime()} - {action} - کاربر: {self.current_user}"
        self.logs.append(log_entry)
        self.save_data()
    
    def init_ui(self):
        self.central_widget = QWidget()
        self.setCentralWidget(self.central_widget)
        self.layout = QVBoxLayout(self.central_widget)
        self.tabs = QTabWidget()
        self.layout.addWidget(self.tabs)
        self.create_main_tabs()
        self.tabs.setCurrentIndex(0)
    
    def create_main_tabs(self):
        self.cut_tab = QWidget()
        self.init_cut_tab()
        self.tabs.addTab(self.cut_tab, "برش زدن")
        
        self.tooling_tab = QWidget()
        self.init_tooling_tab()
        self.tabs.addTab(self.tooling_tab, "ابزار زدن")
        
        self.report_tab = QWidget()
        self.init_report_tab()
        self.tabs.addTab(self.report_tab, "گزارش مالی")
        
        self.inventory_tab = QWidget()
        self.init_inventory_tab()
        self.tabs.addTab(self.inventory_tab, "انبار")
        
        self.backup_tab = QWidget()
        self.init_backup_tab()
        self.tabs.addTab(self.backup_tab, "پشتیبان")
        
        self.logs_tab = QWidget()
        self.init_logs_tab()
        self.tabs.addTab(self.logs_tab, "لاگ‌ها")
    
    def init_cut_tab(self):
        layout = QVBoxLayout()
        self.lbl_datetime = QLabel(get_jalali_datetime())
        layout.addWidget(self.lbl_datetime)
        
        form_layout = QFormLayout()
        customer_layout = QHBoxLayout()
        self.cmb_customer = QComboBox()
        self.btn_add_customer = QPushButton("+")
        self.btn_remove_customer = QPushButton("-")
        
        customer_layout.addWidget(QLabel("نام مشتری:"))
        customer_layout.addWidget(self.cmb_customer)
        customer_layout.addWidget(self.btn_add_customer)
        customer_layout.addWidget(self.btn_remove_customer)
        form_layout.addRow(customer_layout)
        
        self.cmb_stone_type = QComboBox()
        self.cmb_stone_type.addItems(["سنگ", "سرامیک"])
        form_layout.addRow("نوع سنگ:", self.cmb_stone_type)
        
        self.lbl_stone_detail = QLabel("جزئیات سنگ (می‌توانید چند مورد انتخاب کنید):")
        self.list_stone_detail = QListWidget()
        self.list_stone_detail.setSelectionMode(QListWidget.MultiSelection)
        form_layout.addRow(self.lbl_stone_detail)
        form_layout.addRow(self.list_stone_detail)
        
        self.cmb_cut_thickness = QComboBox()
        form_layout.addRow("قطر برش:", self.cmb_cut_thickness)
        
        self.cmb_cut_type = QComboBox()
        self.cmb_cut_type.addItems(["برش معمولی", "برش 45 درجه", "نمره برش"])
        form_layout.addRow("نوع برش:", self.cmb_cut_type)
        
        price_layout = QHBoxLayout()
        self.lbl_price = QLabel("قیمت هر متر:")
        self.txt_price = QLineEdit()
        self.lbl_area = QLabel("متراژ:")
        self.txt_area = QLineEdit()
        
        price_layout.addWidget(self.lbl_price)
        price_layout.addWidget(self.txt_price)
        price_layout.addWidget(self.lbl_area)
        price_layout.addWidget(self.txt_area)
        form_layout.addRow(price_layout)
        
        self.btn_add = QPushButton("افزودن تراکنش")
        self.btn_add.clicked.connect(self.add_cut_transaction)
        layout.addLayout(form_layout)
        layout.addWidget(self.btn_add)
        self.cut_tab.setLayout(layout)
        
        self.cmb_stone_type.currentIndexChanged.connect(self.update_stone_details)
        self.btn_add_customer.clicked.connect(self.add_customer)
        self.btn_remove_customer.clicked.connect(self.remove_customer)
        self.update_stone_details()
        self.update_customer_list()
    
    def update_stone_details(self):
        stone_type = self.cmb_stone_type.currentText()
        self.list_stone_detail.clear()
        self.cmb_cut_thickness.clear()
        
        if stone_type == "سنگ":
            self.list_stone_detail.addItems(["گرانیت", "مرمریت", "تراورتن", "چینی"])
            self.cmb_cut_thickness.addItems(["2 سانت", "3 سانت"])
        elif stone_type == "سرامیک":
            self.list_stone_detail.addItems(["سرامیک", "کاشی", "پرسلان"])
            self.cmb_cut_thickness.addItems(["1 سانت", "2 سانت"])
    
    def update_customer_list(self):
        self.cmb_customer.clear()
        self.cmb_customer.addItems([c['name'] for c in self.customers])
    
    def add_customer(self):
        dialog = CustomerDialog(self)
        if dialog.exec_() == QDialog.Accepted:
            customer_data = dialog.get_customer_data()
            if customer_data['name']:
                self.customers.append(customer_data)
                self.save_data()
                self.update_customer_list()
                self.cmb_customer.setCurrentText(customer_data['name'])
    
    def remove_customer(self):
        customer_name = self.cmb_customer.currentText()
        if customer_name:
            reply = QMessageBox.question(
                self, "حذف مشتری",
                f"آیا از حذف مشتری '{customer_name}' مطمئن هستید؟",
                QMessageBox.Yes | QMessageBox.No
            )
            if reply == QMessageBox.Yes:
                self.customers = [c for c in self.customers if c['name'] != customer_name]
                self.save_data()
                self.update_customer_list()
    
    def add_cut_transaction(self):
        if not self.cmb_customer.currentText():
            QMessageBox.warning(self, "خطا", "لطفاً مشتری را انتخاب کنید!")
            return
        
        selected_stones = [item.text() for item in self.list_stone_detail.selectedItems()]
        if not selected_stones:
            QMessageBox.warning(self, "خطا", "لطفاً حداقل یک نوع سنگ انتخاب کنید!")
            return
        
        try:
            price = float(self.txt_price.text())
            area = float(self.txt_area.text())
        except ValueError:
            QMessageBox.warning(self, "خطا", "قیمت و متراژ باید عدد باشند!")
            return
        
        for stone in selected_stones:
            transaction = {
                "type": "برش زدن",
                "datetime": get_jalali_datetime(),
                "customer": self.cmb_customer.currentText(),
                "stone_type": self.cmb_stone_type.currentText(),
                "stone_detail": stone,
                "cut_thickness": self.cmb_cut_thickness.currentText(),
                "cut_type": self.cmb_cut_type.currentText(),
                "price": price,
                "area": area,
                "total": price * area,
                "paid": False,
                "tool_type": "",
                "tool_size": ""
            }
            self.transactions.append(transaction)
        
        self.txt_price.clear()
        self.txt_area.clear()
        self.save_data()
        self.log_action("افزودن تراکنش برش")
        self.update_report()
        QMessageBox.information(self, "موفق", "تراکنش‌ها با موفقیت ثبت شدند!")
    
    def init_tooling_tab(self):
        layout = QVBoxLayout()
        self.btn_open_tooling_dialog = QPushButton("ثبت ابزار زدن")
        self.btn_open_tooling_dialog.clicked.connect(self.open_tooling_dialog)
        layout.addWidget(self.btn_open_tooling_dialog)
        self.tooling_tab.setLayout(layout)
    
    def open_tooling_dialog(self):
        dialog = ToolingDialog(self)
        dialog.exec_()
    
    def add_transaction(self, transaction):
        self.transactions.append(transaction)
        self.save_data()
        self.log_action("افزودن تراکنش ابزار")
        self.update_report()
    
    def init_report_tab(self):
        layout = QVBoxLayout()
        self.tbl_transactions = QTableWidget()
        self.tbl_transactions.setColumnCount(11)
        self.tbl_transactions.setHorizontalHeaderLabels([
            "تاریخ", "نوع", "مشتری", "نوع سنگ", 
            "جزئیات سنگ", "قطر برش", "نوع برش", 
            "نوع ابزار", "قیمت کل", "متراژ", "وضعیت پرداخت"
        ])
        self.tbl_transactions.horizontalHeader().setSectionResizeMode(QHeaderView.Stretch)
        self.tbl_transactions.setSelectionMode(QTableWidget.MultiSelection)
        layout.addWidget(self.tbl_transactions)
        
        btn_layout = QHBoxLayout()
        self.btn_edit = QPushButton("ویرایش")
        self.btn_delete = QPushButton("حذف")
        self.btn_search = QPushButton("جستجو")
        self.btn_single_report = QPushButton("گزارش مالی تکی")
        self.btn_daily_report = QPushButton("گزارش روزانه")
        
        btn_layout.addWidget(self.btn_edit)
        btn_layout.addWidget(self.btn_delete)
        btn_layout.addWidget(self.btn_search)
        btn_layout.addWidget(self.btn_single_report)
        btn_layout.addWidget(self.btn_daily_report)
        layout.addLayout(btn_layout)
        
        self.chart_view = QChartView()
        self.chart_view.setRenderHint(QPainter.Antialiasing)
        layout.addWidget(self.chart_view)
        self.report_tab.setLayout(layout)
        
        self.btn_edit.clicked.connect(self.edit_transaction)
        self.btn_delete.clicked.connect(self.delete_transaction)
        self.btn_search.clicked.connect(self.search_transactions)
        self.btn_single_report.clicked.connect(self.generate_single_report)
        self.btn_daily_report.clicked.connect(self.generate_daily_report)
        self.update_report()
    
    def update_report(self):
        self.tbl_transactions.setRowCount(len(self.transactions))
        for row, transaction in enumerate(self.transactions):
            datetime_str = transaction.get("datetime", get_jalali_datetime())
            total = transaction.get("total", 0)
            area = transaction.get("area", 0)
            
            self.tbl_transactions.setItem(row, 0, QTableWidgetItem(datetime_str))
            self.tbl_transactions.setItem(row, 1, QTableWidgetItem(transaction.get("type", "")))
            self.tbl_transactions.setItem(row, 2, QTableWidgetItem(transaction.get("customer", "")))
            self.tbl_transactions.setItem(row, 3, QTableWidgetItem(transaction.get("stone_type", "")))
            self.tbl_transactions.setItem(row, 4, QTableWidgetItem(transaction.get("stone_detail", "")))
            self.tbl_transactions.setItem(row, 5, QTableWidgetItem(transaction.get("cut_thickness", "")))
            self.tbl_transactions.setItem(row, 6, QTableWidgetItem(transaction.get("cut_type", "")))
            self.tbl_transactions.setItem(row, 7, QTableWidgetItem(transaction.get("tool_type", "")))
            self.tbl_transactions.setItem(row, 8, QTableWidgetItem(f"{total:,}"))
            self.tbl_transactions.setItem(row, 9, QTableWidgetItem(f"{area:,}"))
            
            status_item = QTableWidgetItem()
            status_item.setCheckState(Qt.Checked if transaction.get("paid", False) else Qt.Unchecked)
            status_item.setFlags(Qt.ItemIsUserCheckable | Qt.ItemIsEnabled)
            self.tbl_transactions.setItem(row, 10, status_item)
        
        self.update_chart()
    
    def update_chart(self):
        chart = QChart()
        chart.setTitle("آمار فروش")
        
        bar_set = QBarSet("فروش روزانه")
        categories = []
        
        sales_by_date = {}
        for t in self.transactions:
            datetime_str = t.get("datetime", get_jalali_datetime())
            date = datetime_str.split("ساعت")[0].strip()
            
            if date not in sales_by_date:
                sales_by_date[date] = 0
            sales_by_date[date] += t.get("total", 0)
        
        for date, total in sales_by_date.items():
            bar_set.append(total)
            categories.append(date)
        
        series = QBarSeries()
        series.append(bar_set)
        chart.addSeries(series)
        
        axis_x = QBarCategoryAxis()
        axis_x.append(categories)
        chart.addAxis(axis_x, Qt.AlignBottom)
        series.attachAxis(axis_x)
        
        axis_y = QValueAxis()
        axis_y.setLabelFormat("%.0f")
        chart.addAxis(axis_y, Qt.AlignLeft)
        series.attachAxis(axis_y)
        
        self.chart_view.setChart(chart)
    
    def edit_transaction(self):
        selected_row = self.tbl_transactions.currentRow()
        if selected_row == -1:
            QMessageBox.warning(self, "خطا", "لطفاً یک تراکنش را انتخاب کنید.")
            return
        
        transaction = self.transactions[selected_row]
        new_customer, ok = QInputDialog.getText(
            self, "ویرایش مشتری", "نام مشتری:", 
            text=transaction.get("customer", "")
        )
        if ok and new_customer:
            transaction["customer"] = new_customer
        
        reply = QMessageBox.question(
            self, "وضعیت پرداخت", 
            "آیا مشتری پرداخت کرده است؟",
            QMessageBox.Yes | QMessageBox.No
        )
        transaction["paid"] = reply == QMessageBox.Yes
        
        self.save_data()
        self.update_report()
        QMessageBox.information(self, "موفق", "تراکنش با موفقیت ویرایش شد.")
    
    def delete_transaction(self):
        selected_rows = set(index.row() for index in self.tbl_transactions.selectedIndexes())
        if not selected_rows:
            QMessageBox.warning(self, "خطا", "لطفاً حداقل یک تراکنش را انتخاب کنید.")
            return
        
        reply = QMessageBox.question(
            self, "حذف تراکنش", 
            f"آیا مطمئن هستید که می‌خواهید {len(selected_rows)} تراکنش را حذف کنید؟",
            QMessageBox.Yes | QMessageBox.No
        )
        
        if reply == QMessageBox.Yes:
            for row in sorted(selected_rows, reverse=True):
                if row < len(self.transactions):
                    self.transactions.pop(row)
            
            self.save_data()
            self.update_report()
            QMessageBox.information(self, "موفق", "تراکنش‌ها با موفقیت حذف شدند.")
    
    def search_transactions(self):
        dialog = SearchDialog(self)
        if dialog.exec_() == QDialog.Accepted:
            search_params = dialog.get_search_params()
            filtered = []
            
            for t in self.transactions:
                match = False
                
                if search_params['field'] == "نام مشتری":
                    match = search_params['value'].lower() in t.get('customer', '').lower()
                elif search_params['field'] == "نوع سنگ":
                    match = search_params['value'].lower() in t.get('stone_type', '').lower()
                elif search_params['field'] == "آدرس":
                    customer = next((c for c in self.customers if c['name'] == t.get('customer', '')), None)
                    if customer and search_params['value'].lower() in customer.get('address', '').lower():
                        match = True
                elif search_params['field'] == "نوع ابزار":
                    match = search_params['value'].lower() in t.get('tool_type', '').lower()
                elif search_params['field'] == "وضعیت پرداخت":
                    match = t.get('paid', False) == search_params['paid_only']
                
                if match:
                    filtered.append(t)
            
            self.tbl_transactions.setRowCount(len(filtered))
            for row, t in enumerate(filtered):
                self.tbl_transactions.setItem(row, 0, QTableWidgetItem(t.get("datetime", get_jalali_datetime())))
                self.tbl_transactions.setItem(row, 1, QTableWidgetItem(t.get("type", "")))
                self.tbl_transactions.setItem(row, 2, QTableWidgetItem(t.get("customer", "")))
                self.tbl_transactions.setItem(row, 3, QTableWidgetItem(t.get("stone_type", "")))
                self.tbl_transactions.setItem(row, 4, QTableWidgetItem(t.get("stone_detail", "")))
                self.tbl_transactions.setItem(row, 5, QTableWidgetItem(t.get("cut_thickness", "")))
                self.tbl_transactions.setItem(row, 6, QTableWidgetItem(t.get("cut_type", "")))
                self.tbl_transactions.setItem(row, 7, QTableWidgetItem(t.get("tool_type", "")))
                self.tbl_transactions.setItem(row, 8, QTableWidgetItem(f"{t.get('total', 0):,}"))
                self.tbl_transactions.setItem(row, 9, QTableWidgetItem(f"{t.get('area', 0):,}"))
                
                status_item = QTableWidgetItem()
                status_item.setCheckState(Qt.Checked if t.get("paid", False) else Qt.Unchecked)
                status_item.setFlags(Qt.ItemIsUserCheckable | Qt.ItemIsEnabled)
                self.tbl_transactions.setItem(row, 10, status_item)
    
    def generate_single_report(self):
        selected_rows = set(index.row() for index in self.tbl_transactions.selectedIndexes())
        if not selected_rows:
            QMessageBox.warning(self, "خطا", "لطفاً حداقل یک تراکنش را انتخاب کنید")
            return
        
        transactions = [self.transactions[row] for row in selected_rows if row < len(self.transactions)]
        self.generate_report(transactions)
    
    def generate_daily_report(self):
        today = get_jalali_datetime().split("ساعت")[0].strip()
        daily_transactions = [t for t in self.transactions if t.get("datetime", "").startswith(today)]
        
        if not daily_transactions:
            QMessageBox.information(self, "اطلاع", "هیچ تراکنشی برای امروز ثبت نشده است.")
            return
        
        self.generate_report(daily_transactions)
    
    def generate_report(self, transactions):
        report_text = "===== گزارش مالی =====\n\n"
        report_text += f"تاریخ گزارش: {get_jalali_datetime()}\n"
        report_text += f"تعداد تراکنش‌ها: {len(transactions)}\n\n"
        
        total_amount = sum(t.get('total', 0) for t in transactions)
        report_text += f"جمع کل: {total_amount:,} ریال\n\n"
        
        report_text += "--- جزئیات تراکنش‌ها ---\n"
        for i, t in enumerate(transactions, 1):
            report_text += f"\nتراکنش {i}:\n"
            report_text += f"  نوع: {t.get('type', '')}\n"
            report_text += f"  مشتری: {t.get('customer', '')}\n"
            
            customer = next((c for c in self.customers if c['name'] == t.get('customer', '')), None)
            if customer:
                report_text += f"  آدرس: {customer.get('address', 'ثبت نشده')}\n"
                report_text += f"  تلفن: {customer.get('phone', 'ثبت نشده')}\n"
            
            if t.get('type') == "برش زدن":
                report_text += f"  نوع سنگ: {t.get('stone_type', '')} - {t.get('stone_detail', '')}\n"
                report_text += f"  قطر برش: {t.get('cut_thickness', '')}\n"
                report_text += f"  نوع برش: {t.get('cut_type', '')}\n"
            else:
                report_text += f"  نوع ابزار: {t.get('tool_type', '')} ({t.get('tool_size', '')})\n"
            
            report_text += f"  متراژ: {t.get('area', 0)}\n"
            report_text += f"  قیمت کل: {t.get('total', 0):,} ریال\n"
            report_text += f"  وضعیت پرداخت: {'پرداخت شده' if t.get('paid', False) else 'پرداخت نشده'}\n"
        
        report_text += "\n\nتذکر:\n"
        report_text += "اجناس فوق تا تسویه حساب کامل، نزد خریدار به صورت امانت می‌ماند.\n"
        report_text += "در صورت عدم پرداخت، مسئولیت هرگونه آسیب یا گم شدن با خریدار است."
        
        dialog = ReportDialog(report_text, self)
        dialog.exec_()
    
    def init_inventory_tab(self):
        layout = QVBoxLayout()
        self.inventory_table = QTableWidget()
        self.inventory_table.setColumnCount(4)
        self.inventory_table.setHorizontalHeaderLabels([
            "نام سنگ", "نوع سنگ", "تعداد", "قیمت هر واحد"
        ])
        self.inventory_table.horizontalHeader().setSectionResizeMode(QHeaderView.Stretch)
        layout.addWidget(self.inventory_table)
        
        btn_layout = QHBoxLayout()
        self.btn_add_item = QPushButton("افزودن آیتم")
        self.btn_edit_item = QPushButton("ویرایش آیتم")
        self.btn_delete_item = QPushButton("حذف آیتم")
        btn_layout.addWidget(self.btn_add_item)
        btn_layout.addWidget(self.btn_edit_item)
        btn_layout.addWidget(self.btn_delete_item)
        layout.addLayout(btn_layout)
        
        self.btn_add_item.clicked.connect(self.add_inventory_item)
        self.btn_edit_item.clicked.connect(self.edit_inventory_item)
        self.btn_delete_item.clicked.connect(self.delete_inventory_item)
        
        self.inventory_tab.setLayout(layout)
        self.update_inventory_table()
    
    def update_inventory_table(self):
        self.inventory_table.setRowCount(len(self.inventory))
        for row, item in enumerate(self.inventory):
            self.inventory_table.setItem(row, 0, QTableWidgetItem(item.get("name", "")))
            self.inventory_table.setItem(row, 1, QTableWidgetItem(item.get("type", "")))
            self.inventory_table.setItem(row, 2, QTableWidgetItem(str(item.get("quantity", 0))))
            self.inventory_table.setItem(row, 3, QTableWidgetItem(f"{item.get('price', 0):,}"))
    
    def add_inventory_item(self):
        name, ok = QInputDialog.getText(self, "افزودن آیتم", "نام سنگ:")
        if not ok or not name:
            return
        
        type_, ok = QInputDialog.getText(self, "افزودن آیتم", "نوع سنگ:")
        if not ok or not type_:
            return
        
        quantity, ok = QInputDialog.getInt(self, "افزودن آیتم", "تعداد:", min=1)
        if not ok:
            return
        
        price, ok = QInputDialog.getDouble(self, "افزودن آیتم", "قیمت هر واحد:", min=0.1)
        if not ok:
            return
        
        self.inventory.append({
            "name": name,
            "type": type_,
            "quantity": quantity,
            "price": price
        })
        self.save_data()
        self.update_inventory_table()
    
    def edit_inventory_item(self):
        selected_row = self.inventory_table.currentRow()
        if selected_row == -1 or selected_row >= len(self.inventory):
            QMessageBox.warning(self, "خطا", "لطفاً یک آیتم را انتخاب کنید.")
            return
        
        item = self.inventory[selected_row]
        name, ok = QInputDialog.getText(
            self, "ویرایش آیتم", "نام سنگ:", 
            text=item.get("name", "")
        )
        if not ok or not name:
            return
        
        type_, ok = QInputDialog.getText(
            self, "ویرایش آیتم", "نوع سنگ:", 
            text=item.get("type", "")
        )
        if not ok or not type_:
            return
        
        quantity, ok = QInputDialog.getInt(
            self, "ویرایش آیتم", "تعداد:", 
            value=item.get("quantity", 1), min=1
        )
        if not ok:
            return
        
        price, ok = QInputDialog.getDouble(
            self, "ویرایش آیتم", "قیمت هر واحد:", 
            value=item.get("price", 0.1), min=0.1
        )
        if not ok:
            return
        
        item.update({
            "name": name,
            "type": type_,
            "quantity": quantity,
            "price": price
        })
        self.save_data()
        self.update_inventory_table()
    
    def delete_inventory_item(self):
        selected_row = self.inventory_table.currentRow()
        if selected_row == -1 or selected_row >= len(self.inventory):
            QMessageBox.warning(self, "خطا", "لطفاً یک آیتم را انتخاب کنید.")
            return
        
        reply = QMessageBox.question(
            self, "حذف آیتم", 
            "آیا مطمئن هستید که می‌خواهید این آیتم را حذف کنید؟",
            QMessageBox.Yes | QMessageBox.No
        )
        
        if reply == QMessageBox.Yes:
            self.inventory.pop(selected_row)
            self.save_data()
            self.update_inventory_table()
    
    def init_backup_tab(self):
        layout = QVBoxLayout()
        self.btn_backup = QPushButton("پشتیبان‌گیری")
        self.btn_restore = QPushButton("بازیابی")
        layout.addWidget(self.btn_backup)
        layout.addWidget(self.btn_restore)
        self.btn_backup.clicked.connect(self.backup_data)
        self.btn_restore.clicked.connect(self.restore_data)
        self.backup_tab.setLayout(layout)
    
    def backup_data(self):
        file_name, _ = QFileDialog.getSaveFileName(
            self, "ذخیره پشتیبان", "", "JSON Files (*.json)"
        )
        if file_name:
            data = {
                "transactions": self.transactions,
                "users": self.users,
                "inventory": self.inventory,
                "logs": self.logs,
                "customers": self.customers
            }
            try:
                with open(file_name, "w", encoding="utf-8") as f:
                    json.dump(data, f, ensure_ascii=False, indent=4)
                QMessageBox.information(self, "موفق", "پشتیبان‌گیری با موفقیت انجام شد.")
            except Exception as e:
                QMessageBox.critical(self, "خطا", f"خطا در ذخیره پشتیبان: {str(e)}")
    
    def restore_data(self):
        file_name, _ = QFileDialog.getOpenFileName(
            self, "بازیابی پشتیبان", "", "JSON Files (*.json)"
        )
        if file_name:
            try:
                with open(file_name, "r", encoding="utf-8") as f:
                    data = json.load(f)
                
                self.transactions = data.get("transactions", [])
                self.users = data.get("users", [])
                self.inventory = data.get("inventory", [])
                self.logs = data.get("logs", [])
                self.customers = data.get("customers", [])
                
                self.update_report()
                self.update_inventory_table()
                self.update_logs()
                
                QMessageBox.information(self, "موفق", "بازیابی با موفقیت انجام شد.")
            except Exception as e:
                QMessageBox.critical(self, "خطا", f"خطا در بازیابی داده‌ها: {str(e)}")
    
    def init_logs_tab(self):
        layout = QVBoxLayout()
        self.logs_text = QTextEdit()
        self.logs_text.setReadOnly(True)
        layout.addWidget(self.logs_text)
        self.logs_tab.setLayout(layout)
        self.update_logs()
    
    def update_logs(self):
        self.logs_text.clear()
        for log in self.logs[-100:]:
            self.logs_text.append(log)
    
    def save_data(self):
        data = {
            "transactions": self.transactions,
            "users": self.users,
            "inventory": self.inventory,
            "logs": self.logs,
            "customers": self.customers
        }
        try:
            with open("data.json", "w", encoding="utf-8") as f:
                json.dump(data, f, ensure_ascii=False, indent=4)
        except Exception as e:
            QMessageBox.critical(self, "خطا", f"خطا در ذخیره داده‌ها: {str(e)}")
    
    def load_data(self):
        try:
            with open("data.json", "r", encoding="utf-8") as f:
                data = json.load(f)
            
            self.transactions = data.get("transactions", [])
            self.users = data.get("users", [])
            self.inventory = data.get("inventory", [])
            self.logs = data.get("logs", [])
            self.customers = data.get("customers", [])
            
            if not self.users:
                self.users = [{"username": "admin", "password": "admin"}]
                self.save_data()
        
        except FileNotFoundError:
            self.users = [{"username": "admin", "password": "admin"}]
            self.save_data()
        except Exception as e:
            QMessageBox.critical(self, "خطا", f"خطا در بارگذاری داده‌ها: {str(e)}")

# ------------------- دیالوگ احراز هویت -------------------
class AuthDialog(QDialog):
    def __init__(self, users, parent=None):
        super().__init__(parent)
        self.setWindowTitle("ورود/ثبت‌نام")
        self.setGeometry(200, 200, 400, 300)
        self.users = users
        self.tabs = QTabWidget()
        self.login_tab = QWidget()
        self.register_tab = QWidget()
        self.tabs.addTab(self.login_tab, "ورود")
        self.tabs.addTab(self.register_tab, "ثبت‌نام")
        layout = QVBoxLayout()
        layout.addWidget(self.tabs)
        self.setLayout(layout)
        self.init_login_tab()
        self.init_register_tab()
    
    def init_login_tab(self):
        layout = QFormLayout()
        self.login_username = QLineEdit()
        self.login_password = QLineEdit()
        self.login_password.setEchoMode(QLineEdit.Password)
        layout.addRow("نام کاربری:", self.login_username)
        layout.addRow("رمز عبور:", self.login_password)
        self.login_btn = QPushButton("ورود")
        self.login_btn.clicked.connect(self.login)
        layout.addRow(self.login_btn)
        self.login_tab.setLayout(layout)
    
    def init_register_tab(self):
        layout = QFormLayout()
        self.reg_username = QLineEdit()
        self.reg_password = QLineEdit()
        self.reg_password.setEchoMode(QLineEdit.Password)
        self.reg_confirm_password = QLineEdit()
        self.reg_confirm_password.setEchoMode(QLineEdit.Password)
        layout.addRow("نام کاربری:", self.reg_username)
        layout.addRow("رمز عبور:", self.reg_password)
        layout.addRow("تکرار رمز عبور:", self.reg_confirm_password)
        self.register_btn = QPushButton("ثبت‌نام")
        self.register_btn.clicked.connect(self.register)
        layout.addRow(self.register_btn)
        self.register_tab.setLayout(layout)
    
    def login(self):
        username = self.login_username.text().strip()
        password = self.login_password.text().strip()
        for user in self.users:
            if user["username"] == username and user["password"] == password:
                self.accept()
                return
        QMessageBox.warning(self, "خطا", "نام کاربری یا رمز عبور اشتباه است!")
    
    def register(self):
        username = self.reg_username.text().strip()
        password = self.reg_password.text().strip()
        confirm = self.reg_confirm_password.text().strip()
        if not username or not password:
            QMessageBox.warning(self, "خطا", "لطفا تمام فیلدها را پر کنید!")
            return
        if password != confirm:
            QMessageBox.warning(self, "خطا", "رمز عبور و تکرار آن مطابقت ندارند!")
            return
        if any(user["username"] == username for user in self.users):
            QMessageBox.warning(self, "خطا", "این نام کاربری قبلا ثبت شده است!")
            return
        self.users.append({"username": username, "password": password})
        QMessageBox.information(self, "موفق", "ثبت‌نام با موفقیت انجام شد!")
        self.accept()

if __name__ == "__main__":
    app = QApplication(sys.argv)
    window = StoneCuttingApp()
    window.show()
    sys.exit(app.exec_())
