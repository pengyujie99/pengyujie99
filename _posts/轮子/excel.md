

配置类

~~~
package com.lichu.marketing.commons.common.utils.excel;

/**
 * excel读取设置
 */
public class ReadConfig {

  /**
   * sheet序号
   */
  private int sheetNumber;

  /**
   * 开始读取的行号
   */
  private int startLineNumber;

  /**
   * 读取结束的行号
   */
  private Integer endLineNumber;

  public ReadConfig(int sheetNumber, int startLineNumber) {
    this.sheetNumber = sheetNumber;
    this.startLineNumber = startLineNumber;
  }

  public ReadConfig(int sheetNumber, int startLineNumber, Integer endLineNumber) {
    this.sheetNumber = sheetNumber;
    this.startLineNumber = startLineNumber;
    this.endLineNumber = endLineNumber;
  }

  public static ReadConfig of(int sheetNumber, int startLineNumber) {
    return new ReadConfig(sheetNumber, startLineNumber);
  }

  public static ReadConfig of(int sheetNumber, int startLineNumber, Integer endLineNumber) {
    return new ReadConfig(sheetNumber, startLineNumber, endLineNumber);
  }

  public int getSheetNumber() {
    return sheetNumber;
  }

  public void setSheetNumber(int sheetNumber) {
    this.sheetNumber = sheetNumber;
  }

  public int getStartLineNumber() {
    return startLineNumber;
  }

  public void setStartLineNumber(int startLineNumber) {
    this.startLineNumber = startLineNumber;
  }

  public Integer getEndLineNumber() {
    return endLineNumber;
  }

  public void setEndLineNumber(Integer endLineNumber) {
    this.endLineNumber = endLineNumber;
  }
}

~~~







excel导入工具方法

~~~xml
package com.lichu.marketing.commons.common.utils;


import com.lichu.marketing.commons.common.exception.BusinessFailedException;
import com.lichu.marketing.commons.common.utils.excel.FieldValidator;
import com.lichu.marketing.commons.common.utils.excel.ReadColumn;
import com.lichu.marketing.commons.common.utils.excel.ReadConfig;
import org.apache.poi.openxml4j.exceptions.InvalidFormatException;
import org.apache.poi.ss.usermodel.*;

import java.io.IOException;
import java.io.InputStream;
import java.lang.reflect.Field;
import java.util.LinkedList;
import java.util.List;

/**
 * excel读取类
 *
 */
public class ExcelReader {

    private final ReadConfig readConfig;

    public ExcelReader(ReadConfig readConfig) {
        this.readConfig = readConfig;
    }

    /**
     *
     * @param inputStream
     * @param modelClass
     * @param maxRow excel最大可导入条数限制，不传不限制
     * @param <T>
     * @return
     * @throws IOException
     * @throws InvalidFormatException
     * @throws IllegalAccessException
     * @throws InstantiationException
     */
    public <T> List<T> read(InputStream inputStream, Class<T> modelClass, Integer maxRow)
            throws IOException, InvalidFormatException, IllegalAccessException, InstantiationException {
        List<T> dataList = new LinkedList<>();
        Field[] fields = modelClass.getDeclaredFields();
        Workbook workbook = WorkbookFactory.create(inputStream);
        Sheet sheet = workbook.getSheetAt(readConfig.getSheetNumber());
        if(null != maxRow && sheet.getPhysicalNumberOfRows() > maxRow){
            throw new BusinessFailedException("最多只可导入"+maxRow+"条数据！");
        }
        int rowCount = readConfig.getEndLineNumber() == null || readConfig.getEndLineNumber() > sheet.getPhysicalNumberOfRows()? sheet.getPhysicalNumberOfRows()
                : readConfig.getEndLineNumber();
        for (int i = readConfig.getStartLineNumber(); i < rowCount; i++) {
            Row row = sheet.getRow(i);
            if(row == null){
                throw new RuntimeException(
                        "第 " + (i + 1) + " 行数据不能为空");
            }
            T entity = modelClass.newInstance();
            for (Field field : fields) {
                if (field.isAnnotationPresent(ReadColumn.class)) {
                    field.setAccessible(true);
                    ReadColumn readColumn = field.getAnnotation(ReadColumn.class);
                    Integer idx = readColumn.value();
                    Cell cell = row.getCell(idx);
                    Object cellValue = null;
                    if (cell != null) {
                        System.out.println("cell.getCellTypeEnum()---------------------------------i:"+idx+"----------:"+cell.getCellTypeEnum());
                        switch (cell.getCellTypeEnum()) {
                            case NUMERIC:
                                cellValue = cell.getNumericCellValue();
                                break;
                            case STRING:
                                cellValue = cell.getStringCellValue();
                                break;
                        }
                    }
                    if (cellValue == null && readColumn.required()) {
                        throw new RuntimeException(
                                "第 " + (i + 1) + " 行，第 " + (idx + 1) + " 列赋值出错：数据不能为空");
                    }
                    FieldValidator fieldValidator = readColumn.validator().newInstance();
                    boolean fieldValid = fieldValidator.validate(cellValue);
                    if (fieldValid) {
                        try {
                            System.out.println("cellValue:"+cellValue);
                            field.set(entity, readColumn.typeHandler().newInstance().convert(cellValue));
                        } catch (Exception e) {
                            throw new RuntimeException(
                                    "第 " + (i + 1) + " 行，第 " + (idx + 1) + " 列赋值出错，单元格格式错误：" + e.getMessage());
                        }
                    } else {
                        throw new RuntimeException(
                                "第 " + (i + 1) + " 行，第 " + (idx + 1) + " 列赋值出错，数据验证失败：" + fieldValidator
                                        .getErrorMessage());
                    }
                }
            }
            dataList.add(entity);
        }
        return dataList;
    }
}

~~~





使用方法

~~~java
ReadConfig readConfig = ReadConfig.of(0, 1, CommonConstant.MAX_PAGE_SIZE);
        ExcelReader excelReader = new ExcelReader(readConfig);
        List<ReqImportTradeserialCouponSubsidy> list_import = excelReader.read(inputStream, ReqImportTradeserialCouponSubsidy.class, CommonConstant.MAX_EXPORT_DATA);
~~~

